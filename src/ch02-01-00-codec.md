You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-2.1)

------------

# Codec

In this chapter, we'll implement a `Codec` that converts a stream of bytes into
a stream of a messages.

## Creating a new project

Let's start by creating a new project (`cargo new --lib rmp-rpc-demo`), and
adding a few dependencies in `Cargo.toml`:

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
```

We'll split the crate in several modules:

```
.
├── Cargo.toml
├── LICENSE-MIT
├── README.md
└── src
   ├── lib.rs
   ├── client.rs
   ├── codec.rs
   ├── errors.rs
   ├── message.rs
   ├── protocol.rs
   └── server.rs
```

Create the files: `touch src/{client.rs,codec.rs,errors.rs,message.rs,protocol.rs,server.rs}`. And in
`src/lib.rs` we'll add the dependencies we'll need for this chapter, and
declare our modules:

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

## Representing messages

The first thing to do, if we want to convert a stream of bytes into a stream of
messages is to implement a data structure that represents a MessagePack-RPC
message.

The [MessagePack-RPC
specifications](https://github.com/msgpack-rpc/msgpack-rpc/blob/master/spec.md)
defines three type of messages: requests, notifications, and responses. Let's
naively translate this to Rust. In `src/message.rs`:

```rust
// src/message.rs

// the specs defines a request as [type, msgid, method, params]
pub struct Request {
    // this field must be 0
    pub type: u32,
    pub id: u32,
    pub method: String,
    // this field is an "array of arbitrary objects"
    pub params: Vec<???>,
}

// the specs defines a response as [type, msgid, method, params]
pub struct Response {
    // this field must be 1
    pub type: u32,
    pub id: u32,
    pub error: Option<u32>,
    // this field is a arbitrary object
    pub result: ???,
}

pub struct Notification {
    // this field must be 2
    pub type: u32,
    pub method: String,
    // an array of arbitrary value
    pub params: Vec<???>,
}
```

This can be improved in multiple ways:

- The spec mentions "arbitrary values", but it's not clear how this translates
  to Rust (hence the `???`). Fortunately `rmpv` does have a type for "arbitrary
  messagepack values":
  [`Value`](https://docs.rs/rmpv/0.4.0/rmpv/enum.Value.html).
- The `type` field seems redundant. The rust type carries the same information
  than the MessagePack-RPC `type` field. We can get rid of it.
- The `Response` type could be improved: we have a `Result` type in Rust, we
  don't need the error field to represent a error. Let's make the `result`
  field a `Result`.
- We need _one_ message type, not three. This can be solved with a `Message`
  enum.

It becomes:

```rust
// src/message.rs

use rmpv::Value;

pub struct Request {
    pub id: u32,
    pub method: String,
    pub params: Vec<Value>,
}

pub struct Response {
    pub id: u32,
    pub result: Result<Value, Value>,
}

pub struct Notification {
    pub method: String,
    pub params: Vec<Value>,
}

pub enum Message {
    Request(Request),
    Response(Response),
    Notification(Notification),
}
```

## Codec

We now need to create a `Codec`. A codec is a type that implements
[`tokio_io::codec::Decoder`](https://docs.rs/tokio-io/0.1.2/tokio_io/codec/trait.Decoder.html)
and
[`tokio_io::codec::Encoder`](https://docs.rs/tokio-io/0.1.2/tokio_io/codec/trait.Encoder.html).

Here is the skeleton of the code, in `src/codec.rs`:

```rust
// src/codec.rs

use std::io;
use tokio_io::codec::{Decoder, Encoder};
use bytes::BytesMut;

use message::Message;

pub struct Codec;

impl Decoder for Codec {
    // We want the decoder to return Message items
    type Item = Message;
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Self::Item>> {
        unimplemented!()
    }
}

impl Encoder for Codec {
    // We want the encoder to encode Message items
    type Item = Message;
    type Error = io::Error;

    fn encode(&mut self, msg: Self::Item, buf: &mut BytesMut) -> io::Result<()> {
        unimplemented!()
    }
}
```

### Encoding

Let's focus on the encoder first. We need to somehow turn a `Message` into a
stream of bytes conform to the specs.
[`rmpv::encode::write_value()`](https://docs.rs/rmpv/0.4.0/rmpv/encode/fn.write_value.html)
looks promising: it converts a `Value` into a stream of bytes and write these
bytes in a writer. All we have to do, is turn our `Message` into a `Value`. We
can implement this directly on `Message`:

```rust
// src/message.rs

use std::convert::From;
use rmpv::{Value, Integer, Utf8String};

impl Message {
    // Turn the message into a MessagePack value
    fn as_value(&self) -> Value {
        match *self {
            Message::Request(Request { id, ref method, ref params}) => {
                Value::Array(vec![
                    Value::Integer(Integer::from(0)),
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
                    Value::Integer(Integer::from(1)),
                    Value::Integer(Integer::from(id)),
                    error,
                    result,
                ])
            }
            Message::Notification(Notification {ref method, ref params}) => {
                Value::Array(vec![
                    Value::Integer(Integer::from(2)),
                    Value::String(Utf8String::from(method.as_str())),
                    Value::Array(params.to_owned()),
                ])
            }
        }
    }
```

Our `Encoder` implementation becomes straightforward:

```rust
impl Encoder for Codec {
    type Item = Message;
    type Error = io::Error;

    fn encode(&mut self, msg: Self::Item, buf: &mut BytesMut) -> io::Result<()> {
        Ok(rmpv::encode::write_value(&mut buf.writer(), &msg.as_value())?)
    }
}
```

### Decoding

The decoder is slightly more complicated.


#### A naive decoder

We'll start by implementing `decode` on `Message` using
[`rmpv::decode`](https://docs.rs/rmpv/0.4.0/rmpv/decode/index.html). Since it's
a little bit long, we'll split it in multiple methods: `Message::decode` only
decodes the message type, and then delegates the rest to `Request::decode`,
`Response::decode` and `Notification::decode`. We'll only show
`Request::decode` to keep the snippets small.

```rust
// src/message.rs

use rmpv::decode;

// ...

impl Message {
    
    // ...

    pub fn decode<R: Read>(rd: &mut R) -> Message {
        let msg = decode::value::read_value(rd)?;
        if let Value::Array(ref array) = msg {
            if array.len() < 3 {
                // notification are the shortest message and have 3 items
                panic!("Message too short");
            }
            if let Value::Integer(msg_type) = array[0] {
                match msg_type.as_u64() {
                    Some(0) => Message::Request(Message::decode(array)?),
                    Some(1) => Message::Response(Message::decode(array)?),
                    Some(2) => Message::Notification(Message::decode(array)?),
                    _ => panic!("Invalid message type);
                }
            } else {
                panic!("Could not decode message type);
            }
        } else {
            panic!("Value is not an array");
        }
    }
}

impl Request {
    fn decode(array: &[Value]) -> Self {
        if array.len() < 4 { panic!("Too short for a request") ; }

        let id = if let Value::Integer(id) = array[1] {
            id.as_u64().and_then(|id| Some(id as u32)).unwrap();
        } else {
            panic!("Cannot decode request ID");
        };

        let method = if let Value::String(ref method) = array[2] {
            method.as_str().and_then(|s| Some(s.to_string())).unwrap();
        } else {
            panic!("Cannot decode request method");
        };

        let params = if let Value::Array(ref params) = array[3] {
            params.clone()
        } else {
            panic!("Cannot decode request parameters");
        };
        Request {id: id, method: method, params: params}
    }
}

// etc. There is a similar decode() implementation for the Notification and Response types.
```

A _very_ naive decoder implementation could be:

```rust
impl Decoder for Codec {
    type Item = Message;
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Self::Item>> {
        let mut buf = io::Cursor::new(&src);
        Ok(Message::decode(&mut buf))
    }
}
```

Unfortunately, this won't work. It won't because we have no way to know if
there are enough bytes to read in the `BytesMut` buffer to decode a full
message. When tokio choses to call `decode`, the buffer may even be empty. And
given our `Message::decode` implementation, this will panic.


#### Adding error handling

If there are not enough bytes to read, we need to let tokio know that we need
more bytes. Tokio we'll re-call the method later, when there is more to read.
We do so by returning `Ok(None)`. But how do we know we need more bytes? Error
handling! `Message::decode` could return a specific error when it fails to
decode a message because it's incomplete. We can create a new `DecodeError`
error type in `src/errors.rs`:

```rust
// src/errors.rs

use std::{error, fmt, io};
use rmpv::decode;

#[derive(Debug)]
pub enum DecodeError {
    // Some bytes are missing to decode a full msgpack value
    Truncated,

    // A byte sequence could not be decoded as a msgpack value, or this value is not a valid
    // msgpack-rpc message.
    Invalid,

    // An unknown IO error while reading a byte sequence
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

            // If this occurs, it means `rmpv` was unable to read enough bytes to decode a full
            // MessagePack value, so we convert this error in a DecodeError::Truncated
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

With error handling, we can clean up our `decode` methods:

```rust
// src/message.rs

impl Message {
    pub fn decode<R: Read>(rd: &mut R) -> Result<Message, DecodeError> {
        let msg = decode::value::read_value(rd)?;
        if let Value::Array(ref array) = msg {
            if array.len() < 3 { return Err(DecodeError::Invalid); }
            if let Value::Integer(msg_type) = array[0] {
                match msg_type.as_u64() {
                    Some(0) => return Ok(Message::Request(Request::decode(array)?)),
                    Some(1) => return Ok(Message::Response(Response::decode(array)?)),
                    Some(2) => return Ok(Message::Notification(Notification::decode(array)?)),
                    _ => return Err(DecodeError::Invalid),
                }
            } else {
                return Err(DecodeError::Invalid);
            }
        } else {
            return Err(DecodeError::Invalid);
        }
    }
    // ...
}
 
// We only show the implementation for Request. It is similar for Notification
// and Response

impl Request {
    fn decode(array: &[Value]) -> Result<Self, DecodeError> {
        if array.len() < 4 { return Err(DecodeError::Invalid); }

        let id = if let Value::Integer(id) = array[1] {
            id.as_u64().and_then(|id| Some(id as u32)).ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        let method = if let Value::String(ref method) = array[2] {
            method.as_str().and_then(|s| Some(s.to_string())).ok_or(DecodeError::Invalid)?
        } else {
            return Err(DecodeError::Invalid);
        };

        let params = if let Value::Array(ref params) = array[3] {
            params.clone()
        } else {
            return Err(DecodeError::Invalid);
        };

        Ok(Request {id: id, method: method, params: params})
    }
}
```

#### Final decoder

The actual `Decoder` implementation now becomes:

```rust
impl Decoder for Codec {
    type Item = Message;
    type Error = io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> io::Result<Option<Self::Item>> {
        let res: Result<Option<Self::Item>, Self::Error>;

        // We keep track of how many bytes we read
        let position = {

            // We wrap the buffer into a Cursor that counts how many bytes we read
            let mut buf = io::Cursor::new(&src);

            loop {
                // Try to decode a message
                match Message::decode(&mut buf) {
                    // We got a message, so we break out of the loop
                    Ok(message) => {
                        res = Ok(Some(message));
                        break;
                    }
                    Err(err) => {
                        match err {
                            // Not enough bytes to decode a full message. Return Ok(None) to tell
                            // tokio to retry later when there is more to read
                            DecodeError::Truncated => return Ok(None),
                            // We decoded a MessagePack value, but it's not a valid message,
                            // so we go on, and try to read another value.
                            DecodeError::Invalid => continue,
                            // Something went wrong, but we don't know why.
                            // It's safer to return an error
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
        // Remove the bytes we read from the buffer
        let _ = src.split_to(position);
        // Return the message (or the error if any)
        res
    }
}
```

I hope the comments are detailed enough to understand what happens. To sum up,
the decoder is called regularly by tokio, each time with three possible
outcomes:

- We are able to read a message => return `Ok(message)` and remove these bytes
  from the buffer.
- There are not enough byte to read a message => return `Ok(None)` to tell
  tokio to retry when there is more to read, and leave the buffer intact.
- An error occurs:
    - An invalid MessagePack value is read => try to read the next one
    - An unknown io error occurs => return `Err(the_error)`
