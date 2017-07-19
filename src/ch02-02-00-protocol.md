# Protocol

In this chapter, we'll see how to use `tokio-proto` to implement our
MessagePack-RPC protocol.

## Identifying the protocol

As explained in [chapter 1.2](ch01-02-tokio.md), a typical tokio client
or server is made of three pieces: a codec, a protocol and a service. The next
step is to implement the protocol.  `tokio-proto` provides implementations for
different types of protocols. As explained in the
[documentation](https://docs.rs/tokio-proto/) it distinguishes:

- **pipelined** and **multiplexed** protocols: a pipelined protocol is a protocol
  where requests are answered in the order in which they are received. A
  multiplexed protocol is a protocol for which it does not matter in which
  order requests are answered. However, that means that the client must be able
  to tell which reponse correspond to which request. For this purpose, requests
  carry an ID, and responses carry the ID of the request they correspond to.
- **streaming** and **non-streaming** protocol: in a streaming protocol, requests
  and responses can start being processed before it is entirely received. A
  non-streaming protocol is a protocol in which requests and responses must be
  completely received before being processed.

MessagePack-RPC is clearly a non-streaming protocol. But is it multiplexed?
Requests and responses have an ID, but notifications don't. So should we use
`tokio_proto::multiplex` or `tokio_proto::pipeline`? Well, none of them
actually. I [asked the
question](https://github.com/tokio-rs/tokio-proto/issues/170) and the advice is
that custom protocols should not be implemented with `tokio-proto`.

For the sake of prototyping though, we'll just ignore notifications and come
back to them later. Then, MessagePack-RPC becomes a true multiplexed protocol,
and we can use
[`tokio_proto::multiplex::ClientProto`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/multiplex/trait.ClientProto.html)
and
[`tokio_proto::multiplex::ServerProto`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/multiplex/trait.ServerProto.html)
traits.

## Implementation

First, we'll need to add `tokio-proto` to our dependencies.

In `Cargo.toml`:

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

And in `src/lib.rs`:

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

The protocol implemententation is straightforward, both for the client and for
the server. All we have to do, is tell the protocol which codec it should use.
Hint: it's the one we wrote in the previous post.

```rust
// src/protocol.rs

use tokio_proto::multiplex::{ClientProto, ServerProto};
use tokio_io::codec::Framed;

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

The code is short, but many things happen:

- We set the `Transport` associated type to [`Framed<T,
  Codec>`](https://docs.rs/tokio-io/0.1.2/tokio_io/codec/struct.Framed.html),
  where `T` is a type that implements `tokio::io::AsyncRead` and
  `tokio::io::AsyncWrite`. `T` can be seen as a raw I/O object, such as a
  TCP/UDP or TLS stream. It manipulates bytes. `Framed<T, Codec>` is a higher
  level I/O objects that uses the `Codec` we wrote earlier to convert this
  stream of bytes into a stream of `Message`.
- We set the `Request` and `Response` associated types to `Message` because
  that's our `Codec` manipulates (it encodes `Message`s and decodes `Message`s)
- I'm less sure about the `BindTransport` associated types. It looks like
  implementation detail to me.

Looks good but when we compile we get a bunch of errors.

```
error[E0271]: type mismatch resolving `<tokio_io::codec::Framed<T, Codec> as futures::stream::Stream>::Item == (u64, Message)`
   --> src/lib.rs:269:43
    |                                                                                                                         
269 | impl<T: AsyncRead + AsyncWrite + 'static> ServerProto<T> for Protocol {
    |                                           ^^^^^^^^^^^^^^ expected enum `Message`, found tuple                           
    |                                                                                                                         
    = note: expected type `Message`                                                                                           
               found type `(u64, Message)`                                                                                    
    = note: required by `tokio_proto::multiplex::ServerProto`                                                                 
```

Indeed, if we [take a closer look]((https://docs.rs/tokio-proto/0.1.1/tokio_proto/multiplex/trait.ClientProto.html)
at
[`tokio_proto::multiplex::ClientProto`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/multiplex/trait.ClientProto.html)
we notice that `Transport` has the following trait bound:

```rust
# // add dummy trait to have some syntax highlighting
# trait Dummy {
type Transport: 'static
              + Stream<Item = (RequestId, Self::Response), Error = io::Error>
              + Sink<SinkItem = (RequestId, Self::Request), SinkError = io::Error>
# }
```

But as per [the `Framed` documention](https://docs.rs/tokio-io/0.1.2/tokio_io/codec/struct.Framed.html),
our `Framed<T, Codec>` implements:

```rust
Stream<Item=Codec::Item, Codec::Error> + Sink<SinkItem = Codec::Item, SinkError = Codec::Error>
```

which corresponds to:

```rust
Stream<Item = Message, Error = io::Error> + Sink<SinkItem = Message, SinkError = io::Error>
```

This is because we're using the multiplexed version of `ClientProto` and
`ServerProto`. The codec needs to be updated to handle the message IDs. Again,
this is pretty easy _if we ignore notifications_.


## Tweaking the codec

The decoder should now return a tuple `(u64, Message)` instead of a `Message`.
We can tweak the `Decodec` implementation to achieve this:

```rust
// src/codec.rs

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
                            // We now need to extract the ID of the message
                            // and return it separately
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
```

Similarly, the `Encoder` should now take an ID as argument. It becomes:

```rust
// src/codec.rs

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

`cargo build`, and... it compiles!
