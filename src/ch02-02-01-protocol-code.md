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
extern crate tokio_proto;

mod client;
mod codec;
mod errors;
mod server;

pub mod message;
pub mod protocol;
```

## codec.rs

```rust
use std::io;

use bytes::{BytesMut, BufMut};
use rmpv;
use tokio_io::codec::{Encoder, Decoder};

use errors::DecodeError;
use message::{Message, Request, Response};

pub struct Codec;

impl Decoder for Codec {
    type Item = (u64, Message);
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Self::Item>> {
        let res: Result<Option<Self::Item>, Self::Error>;
        let position = {
            let mut buf = io::Cursor::new(&src);
            loop {
                match Message::decode(&mut buf) {
                    Ok(message) => {
                        res = match message {
                            Message::Request(Request { id, .. }) | Message::Response(Response { id, .. }) => {
                                Ok(Some((id as u64, message)))
                            },
                            Message::Notification(_) => panic!("Notifications not supported"),
                        };
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
    type Item = (u64, Message);
    type Error = io::Error;

    fn encode(&mut self, item: Self::Item, buf: &mut BytesMut) -> io::Result<()> {
        let (id, mut message) = item;
        match message {
            Message::Response(ref mut response) => {
                response.id = id as u32;
            }
            Message::Request(ref mut request) => {
                request.id = id as u32;
            }
            Message::Notification(_) => panic!("Notifications not supported"),
        }
        Ok(rmpv::encode::write_value(&mut buf.writer(), &message.as_value())?)
    }
}
```

## protocol.rs

```rust
use std::io;

use tokio_proto::multiplex::{ClientProto, ServerProto};
use tokio_io::{AsyncRead, AsyncWrite};
use tokio_io::codec::Framed;

use message::Message;
use codec::Codec;

pub struct Protocol;

impl<T: AsyncRead + AsyncWrite + 'static> ServerProto<T> for Protocol {
    type Transport = Framed<T, Codec>;
    type Request = Message;
    type Response = Message;
    type BindTransport = Result<Self::Transport, io::Error>;
    fn bind_transport(&self, io: T) -> Self::BindTransport {
        Ok(io.framed(Codec))
    }
}

impl<T: AsyncRead + AsyncWrite + 'static> ClientProto<T> for Protocol {
    type Request = Message;
    type Response = Message;
    type Transport = Framed<T, Codec>;
    type BindTransport = Result<Self::Transport, io::Error>;
    fn bind_transport(&self, io: T) -> Self::BindTransport {
        Ok(io.framed(Codec))
    }
}
```
