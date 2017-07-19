You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-2.3.2)

------------

# Wrapping the server

The previous example was quite tedious to write. In this chapter, we'll see
that we can provide some code in our library, that would make it easier to
implement servers. Basically, we are going to move some code from our example
into the library.

## Preliminaries

We'll start by moving the `[dev-dependencies]` we used for our example as
regular dependencies.

In `Cargo.toml`:

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
tokio-proto = "0.1"
tokio-service = "0.1"
```

And the `src/lib.rs` should look like this (note that we also made the
`protocol` module private, and the `client` and `server` modules public):

```rust
extern crate bytes;
extern crate futures;
extern crate rmpv;
extern crate tokio_core;
extern crate tokio_io;
extern crate tokio_proto;
extern crate tokio_service;

mod codec;
mod errors;
mod protocol;

pub mod client;
pub mod message;
pub mod server;
```

## Implementation

In the previous example, our `Service` implementation was only matching on the
incoming messages, and dispatching the requests to the `CalculatorService`
implementation. We could spare users the pain to write this boilerplate code by
wrapping the `Service`, and providing another trait.

Basically, we would ask users to implement this trait:

```rust
use rmpv::Value;

pub trait Handler {
    fn handle_request(&mut self, method: &str, params: &[Value]) -> Result<Value, Value>;
}
```

If we don't want users to implement `Service` themself, we need to provide an
implementation of it for any type that implements our `Handler` trait:

```rust
use futures::Future;
use tokio_service::Service;
use message::Message;

impl<T: Handler> Service for T {
    type Request = Message;
    type Response = Message;
    type Error = io::Error;
    type Future = BoxFuture<Self::Response, Self::Error>;

    fn call(&self, message: Self::Request) -> Self::Future {
        // TODO
    }
}
```

And the actual implementation can amost be copy/pasted from the previous
example:

```rust
use futures::{future, Future};
use tokio_service::Service;
use message::{Request, Response, Message};

impl<T: Handler> Service for T {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Future = Box<Future<Item = Message, Error = ServiceError>>;

    fn call(&self, message: Message) -> Self::Future {
        match req {
            Message::Request( Request { method, params, .. }) => {
                let result = self.handle_request(&method, &params);
                let response = Message::Response(Response { id: 0, result: result });
                return Box::new(future::ok(response));
            }
            _ => Box::new(future::err("Unsupported message type".into())),
        }
    }
}
```

The only thing missing is the `ServiceError` type, but again, we can just copy
paste it from our example. Let's put it under `src/errors.rs`:

```rust
// src/errors.rs

#[derive(Debug)]
pub struct ServiceError(String);

impl fmt::Display for ServiceError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, "ServiceError({})", self.0)
    }
}

impl error::Error for ServiceError {
    fn description(&self) -> &str {
        "An error occured while processing a request"
    }

    fn cause(&self) -> Option<&error::Error> {
        None
    }
}

impl<'a> From<&'a str> for ServiceError {
    fn from(err: &'a str) -> Self {
        ServiceError(err.into())
    }
}

impl From<ServiceError> for io::Error {
    fn from(err: ServiceError) -> Self {
        io::Error::new(io::ErrorKind::Other, err.0)
    }
}
```

Unfortunately, compilation fails:

```
compiling rmp-rpc-demo v0.1.0 (file:///home/corentih/rust/rmp-rpc-demo)
error[E0119]: conflicting implementations of trait `tokio_service::Service` for type `std::boxed::Box<_>`:
  --> src/server.rs:20:1
   |
20 | / impl<T: Handler> Service for T {
21 | |     type Request = Message;
22 | |     type Response = Message;
23 | |     type Error = ServiceError;
...  |
35 | |     }
36 | | }
   | |_^
   |
   = note: conflicting implementation in crate `tokio_service`

error[E0210]: type parameter `T` must be used as the type parameter for some local type (e.g. `MyStruct<T>`); only traits defined in the current crate can be implemented for a type parameter
  --> src/server.rs:20:1
   |
20 | / impl<T: Handler> Service for T {
21 | |     type Request = Message;
22 | |     type Response = Message;
23 | |     type Error = ServiceError;
...  |
35 | |     }
36 | | }
   | |_^

error: aborting due to 2 previous errors

error: Could not compile `rmp-rpc-demo`.

To learn more, run the command again with --verbose.
```

Since `Service` is defined in another crate, we cannot implement it for _any_
trait. That is the priviledge of the crate that defines the trait. The rule
that enforces this is called the
[orphaned rule](https://github.com/rust-lang/rfcs/blob/master/text/1023-rebalancing-coherence.md).
To work around this limitation, we can introduce a new type that is generic
over any type `T` that implement `Handler`:


```rust
pub struct Server<T: Handler>(T);

impl<T: Handler> Server<T> {
    pub fn new(handler: T) -> Self {
        Server(handler)
    }
}

impl<T: Handler> Service for Server<T> {
    // unchanged
}
```

We forgot one last thing: `Server<T>` must not only implement `Service`, but
also `NewService`. In our example, we implemented `NewService` by cloning the
service. We can do the same here and clone `Server<T>`. That means `T` must be
clone-able, so we also add a `Clone` trait bound:

```rust
pub struct Server<T: Handler + Clone>(T);

impl<T: Handler + Clone> Server<T> {
    pub fn new(handler: T) -> Self {
        Server(handler)
    }
}

impl<T: Handler + Clone> Service for Server<T> {
    // unchanged
}

impl<T: Handler + Clone> NewService for Server<T> {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Instance = Server<T>;

    fn new_service(&self) -> io::Result<Self::Instance> {
        Ok(Server(self.0.clone()))
    }
}
```

Finally, we can spare users the effort of starting a new `TcpServer` like we did in the example with:

```rust
use std::thread;
use tokio_proto::TcpServer;
use rmp_rpc_demo::protocol::Protocol;

// ...

fn main() {
    // ...
    let server = TcpServer::new(Protocol, addr);
    thread::spawn(move || {
        server.serve(CalculatorService {});
    });
    // ...
}
```

Let's implement a `serve()` method on `Server<T>` that consumes it:

```rust
use tokio_proto::TcpServer;
use protocol::Protocol;


impl<T: Handler + Clone> Server<T> {
    // ...
    pub fn serve(self, address: SocketAddr) {
        TcpServer::new(Protocol, address).serve(self)
    }
}
```

Not that this is a blocking method. Users will still have to spawn it in a
separate thread to run it in background.

Unfortunately, the code fails to compile:

```
error[E0277]: the trait bound `T: std::marker::Send` is not satisfied in `server::Server<T>`
  --> src/server.rs:25:43
   |
25 |         TcpServer::new(Protocol, address).serve(self)
   |                                           ^^^^^ within `server::Server<T>`, the trait `std::marker::Send` is not implemented for `T`
   |
   = help: consider adding a `where T: std::marker::Send` bound
   = note: required because it appears within the type `server::Server<T>`
```

Indeed, looking at [the documentation for
`TcpServer::serve()`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/struct.TcpServer.html#method.serve),
it appears that the argument has the following trait bounds:

```
NewService + Send + Sync + 'static
```

Well, let's just add these trait bounds:

```rust
pub struct Server<T: Handler + Clone + Sync + Send + 'static>(T);

impl<T: Handler + Clone + Sync + Send + 'static> Server<T> {
    // ...
}

impl<T: Handler + Clone + Sync + Send + 'static> Service for Server<T> {
    // ...
}

impl<T: Handler + Clone + Sync + Send + 'static> NewService for Server<T> {
    // ...
}
```

And this time, it should compile!

# Complete code

## server.rs

```rust
use std::net::SocketAddr;
use std::io;

use futures::{Future, future};
use rmpv::Value;
use tokio_proto::TcpServer;
use tokio_service::{NewService, Service};

use errors::ServiceError;
use message::{Message, Response, Request};
use protocol::Protocol;

pub trait Handler {
    fn handle_request(&self, method: &str, params: &[Value]) -> Result<Value, Value>;
}

pub struct Server<T: Handler + Clone + Sync + Send + 'static>(T);

impl<T: Handler + Clone + Sync + Send + 'static> Server<T> {
    pub fn new(handler: T) -> Self {
        Server(handler)
    }

    pub fn serve(self, address: SocketAddr) {
        TcpServer::new(Protocol, address).serve(self)
    }
}

impl<T: Handler + Clone + Sync + Send + 'static> Service for Server<T> {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Future = Box<Future<Item = Message, Error = ServiceError>>;

    fn call(&self, message: Message) -> Self::Future {
        match message {
            Message::Request( Request { method, params, .. }) => {
                let result = self.0.handle_request(&method, &params);
                let response = Message::Response(Response { id: 0, result: result });
                return Box::new(future::ok(response));
            }
            _ => Box::new(future::err("Unsupported message type".into())),
        }
    }
}

impl<T: Handler + Clone + Sync + Send + 'static> NewService for Server<T> {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Instance = Server<T>;

    fn new_service(&self) -> io::Result<Self::Instance> {
        Ok(Server(self.0.clone()))
    }
}
```
