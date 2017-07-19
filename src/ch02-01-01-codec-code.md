# Complete code

## Cargo.toml

```
[package]
name = "rmp-rpc-demo"
version = "0.0.1"
authors = ["You <you@example.com>"]
description = "a msgpack-rpc client and server based on tokio"

[dependencies]
bytes = "0.4"
rmpv = "0.4"
tokio-io = "0.1"
tokio-proto = "0.1"
```

## lib.rs

```rust
extern crate bytes;
extern crate rmpv;
extern crate tokio_io;

mod client;
mod codec;
mod errors;
mod server;

pub mod message;
pub mod protocol;
```

## message.rs

```rust
use std::convert::From;
use std::io::Read;

use errors::*;
use rmpv::{Value, Integer, Utf8String, decode};

#[derive(PartialEq, Clone, Debug)]
pub enum Message {
    Request(Request),
    Response(Response),
    Notification(Notification),
}

#[derive(PartialEq, Clone, Debug)]
pub struct Request {
    pub id: u32,
    pub method: String,
    pub params: Vec<Value>,
}

#[derive(PartialEq, Clone, Debug)]
pub struct Response {
    pub id: u32,
    pub result: Result<Value, Value>,
}

#[derive(PartialEq, Clone, Debug)]
pub struct Notification {
    pub method: String,
    pub params: Vec<Value>,
}

const REQUEST_MESSAGE: u64 = 0;
const RESPONSE_MESSAGE: u64 = 1;
const NOTIFICATION_MESSAGE: u64 = 2;

impl Message {
    pub fn decode<R>(rd: &mut R) -> Result<Message, DecodeError>
    where
        R: Read,
    {
        let msg = decode::value::read_value(rd)?;
        if let Value::Array(ref array) = msg {
            if array.len() < 3 {
                // notification are the shortest message and have 3 items
                return Err(DecodeError::Invalid);
            }
            if let Value::Integer(msg_type) = array[0] {
                match msg_type.as_u64() {
                    Some(REQUEST_MESSAGE) => {
                        return Ok(Message::Request(Request::decode(array)?));
                    }
                    Some(RESPONSE_MESSAGE) => {
                        return Ok(Message::Response(Response::decode(array)?));
                    }
                    Some(NOTIFICATION_MESSAGE) => {
                        return Ok(Message::Notification(Notification::decode(array)?));
                    }
                    _ => {
                        return Err(DecodeError::Invalid);
                    }
                }
            } else {
                return Err(DecodeError::Invalid);
            }
        } else {
            return Err(DecodeError::Invalid);
        }
    }

    pub fn as_value(&self) -> Value {
        match *self {
            Message::Request(Request {
                id,
                ref method,
                ref params,
            }) => {
                Value::Array(vec![
                    Value::Integer(Integer::from(REQUEST_MESSAGE)),
                    Value::Integer(Integer::from(id)),
                    Value::String(Utf8String::from(method.as_str())),
                    Value::Array(params.clone()),
                ])
            }
            Message::Response(Response { id, ref result }) => {
                let (error, result) = match *result {
                    Ok(ref result) => (Value::Nil, result.to_owned()),
                    Err(ref err) => (err.to_owned(), Value::Nil),
                };
                Value::Array(vec![
                    Value::Integer(Integer::from(RESPONSE_MESSAGE)),
                    Value::Integer(Integer::from(id)),
                    error,
                    result,
                ])
            }
            Message::Notification(Notification {
                ref method,
                ref params,
            }) => {
                Value::Array(vec![
                    Value::Integer(Integer::from(NOTIFICATION_MESSAGE)),
                    Value::String(Utf8String::from(method.as_str())),
                    Value::Array(params.to_owned()),
                ])
            }
        }
    }
}

impl Notification {
    fn decode(array: &[Value]) -> Result<Self, DecodeError> {
        if array.len() < 3 {
            return Err(DecodeError::Invalid);
        }

        let method = if let Value::String(ref method) = array[1] {
            method
                .as_str()
                .and_then(|s| Some(s.to_string()))
                .ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        let params = if let Value::Array(ref params) = array[2] {
            params.clone()
        } else {
            return Err(DecodeError::Invalid);
        };

        Ok(Notification {
            method: method,
            params: params,
        })
    }
}

impl Request {
    fn decode(array: &[Value]) -> Result<Self, DecodeError> {
        if array.len() < 4 {
            return Err(DecodeError::Invalid);
        }

        let id = if let Value::Integer(id) = array[1] {
            id.as_u64()
                .and_then(|id| Some(id as u32))
                .ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        let method = if let Value::String(ref method) = array[2] {
            method
                .as_str()
                .and_then(|s| Some(s.to_string()))
                .ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        let params = if let Value::Array(ref params) = array[3] {
            params.clone()
        } else {
            return Err(DecodeError::Invalid);
        };

        Ok(Request {
            id: id,
            method: method,
            params: params,
        })
    }
}

impl Response {
    fn decode(array: &[Value]) -> Result<Self, DecodeError> {
        if array.len() < 2 {
            return Err(DecodeError::Invalid);
        }

        let id = if let Value::Integer(id) = array[1] {
            id.as_u64()
                .and_then(|id| Some(id as u32))
                .ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        match array[2] {
            Value::Nil => {
                Ok(Response {
                    id: id,
                    result: Ok(array[3].clone()),
                })
            }
            ref error => {
                Ok(Response {
                    id: id,
                    result: Err(error.clone()),
                })
            }
        }
    }
}
```

## codec.rs

```rust
use std::io;

use bytes::{BytesMut, BufMut};
use rmpv;
use tokio_io::codec::{Encoder, Decoder};

use errors::DecodeError;
use message::Message;

pub struct Codec;

impl Decoder for Codec {
    type Item = Message;
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Self::Item>> {
        let res: Result<Option<Self::Item>, Self::Error>;
        let position = {
            let mut buf = io::Cursor::new(&src);
            loop {
                match Message::decode(&mut buf) {
                    Ok(message) => {
                        res = Ok(Some(message));
                        break;
                    }
                    Err(err) => {
                        match err {
                            DecodeError::Truncated => return Ok(None),
                            DecodeError::Invalid => continue,
                            DecodeError::UnknownIo(io_err) => {
                                res = Err(io_err);
                                break;
                            }
                        }
                    }
                }
            }
            buf.position() as usize
        };
        let _ = src.split_to(position);
        res
    }
}

impl Encoder for Codec {
    type Item = Message;
    type Error = io::Error;

    fn encode(&mut self, msg: Self::Item, buf: &mut BytesMut) -> io::Result<()> {
        Ok(rmpv::encode::write_value(
            &mut buf.writer(),
            &msg.as_value(),
        )?)
    }
}
```

## errors.rs

```rust
use std::{error, fmt, io};

use rmpv::decode;

#[derive(Debug)]
pub enum DecodeError {
    Truncated,
    Invalid,
    UnknownIo(io::Error),
}

impl fmt::Display for DecodeError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        error::Error::description(self).fmt(f)
    }
}

impl error::Error for DecodeError {
    fn description(&self) -> &str {
        match *self {
            DecodeError::Truncated => "could not read enough bytes to decode a complete message",
            DecodeError::UnknownIo(_) => "Unknown IO error while decoding a message",
            DecodeError::Invalid => "the byte sequence is not a valid msgpack-rpc message",
        }
    }

    fn cause(&self) -> Option<&error::Error> {
        match *self {
            DecodeError::UnknownIo(ref e) => Some(e),
            _ => None,
        }
    }
}

impl From<io::Error> for DecodeError {
    fn from(err: io::Error) -> DecodeError {
        match err.kind() {
            io::ErrorKind::UnexpectedEof => DecodeError::Truncated,
            io::ErrorKind::Other => {
                if let Some(cause) = err.get_ref().unwrap().cause() {
                    if cause.description() == "type mismatch" {
                        return DecodeError::Invalid;
                    }
                }
                DecodeError::UnknownIo(err)
            }
            _ => DecodeError::UnknownIo(err),

        }
    }
}

impl From<decode::Error> for DecodeError {
    fn from(err: decode::Error) -> DecodeError {
        match err {
            decode::Error::InvalidMarkerRead(io_err) |
            decode::Error::InvalidDataRead(io_err) => From::from(io_err),
        }
    }
}
```
