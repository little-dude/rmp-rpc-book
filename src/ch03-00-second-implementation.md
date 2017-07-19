# Second implementation

Using the full tokio stack, we built a basic MessagePack-RPC client and server
able to handle requests and responses, but unable to handle notifications. This
limitation is due to `tokio-proto` only providing protocol implementations for
request/response based protocols. The problem is that, there is no way to
extend `tokio-proto`, so we will have to re-implement something ourself.

We'll start by the server (the easy part), then implement a client. The
implementation of both component is heavily inspired by Tokio's `tokio-proto`
and `tokio-service`.

## Preliminaries

Before starting implementing the server, let's cleanup a few things:

- remove the `protocol` module:
```
rm -f src/protocol.rs
sed -i '/mod protocol;/d' src/lib.rs
```
- empty the `server` and `client` modules:
```
printf '' | tee src/{server.rs,client.rs}
```
- remove the `tokio_proto` and `tokio_service` dependencies:
```
sed -i '/tokio[-_]\(service\|proto\)/d' Cargo.toml src/lib.rs
```
- remove the "hack" in the codec that we introduced in the [chapter
  2.2](ch02-02-00-protocol.md), when we wanted the codec to handle IDs. This is
  not necessary anymore since we won't rely on `tokio_proto`. The code should be the same than at the [end of chapter 2.1](ch02-01-01-codec-code.html#codecrs).


## Complete code

### Cargo.toml

```
[package]
name = "rmp-rpc-demo"
version = "0.0.1"
authors = ["You <you@example.com>"]
description = "a msgpack-rpc client and server based on tokio"

[dependencies]
bytes = "0.4"
futures = "0.1"
rmpv = "0.4"
tokio-core = "0.1"
tokio-io = "0.1"
```

### src/lib.rs

```rust
extern crate bytes;
extern crate futures;
extern crate rmpv;
extern crate tokio_core;
extern crate tokio_io;

mod codec;
mod errors;

pub mod client;
pub mod message;
pub mod server;
```

### src/client.rs

```rust
// empty
```

### src/server.rs

```rust
// empty
```

### codec.rs

```rust
// src/codec.rs

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
