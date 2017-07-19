You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-2.3.1)

------------

# First example

In this part, we'll build a simple example server and make a few requests.

## Specifications

The server we are going to build provides two methods `add`, and `sub`, that
takes two integers as parameters, and return respectively their sum and their
difference. I does not handle any notifications.

## Creating an example

Rust makes it pretty easy to create example. We are going to put all the code under `examples/calculator.rs`:

```
mkdir examples
touch examples/calculator.rs
```

We'll need a few additional dependencies that we declare them in `Cargo.toml`:

```
[dev-dependencies]
tokio-core = "0.1"
tokio-service = "0.1"
futures = "0.1"
```

To make things easier, let also import everything we're going to need:

```rust
// examples/calculator.rs

extern crate futures;
extern crate rmpv;
extern crate rmp_rpc;
extern crate tokio_core;
extern crate tokio_proto;
extern crate tokio_service;

use std::{error, io, fmt, thread};
use std::time::Duration;

use tokio_core::reactor::Core;
use tokio_proto::{TcpClient, TcpServer};
use tokio_service::{NewService, Service};
use rmpv::Value;
use futures::{future, Future};
use rmp_rpc::{Message, Response, Request, Protocol};

fn main() {
}
```

## Server implementation

[As explained](https://docs.rs/tokio-service/0.1.0/tokio_service/trait.Service.html)
in the `Service` trait documentation, a server implements the `Service` trait.
We'll create an empty `CalculatorService` type, and implement `Service` for it:

```rust
struct CalculatorService;

impl Service for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = String;
    type Future = Box<Future<Item = Message, Error = String>>;

    fn call(&self, req: Message) -> Self::Future {
        unimplemented!()
    }
}
```

This little snippet already deserves some explanations:

- since our protocol encodes and decodes `Message`s, the `Service::Request` and `Service::Response` associated types are both `Message`.
- for now, the `Service::Error` type does not really matter. We used a `String` because that's the most simple type of error.
- the trait imposes that the service returns a type that implements `Future<Item=Self::Message, Error=Self::Error>`. The easiest way to return a `Future` at the moment is to use [trait objects](https://doc.rust-lang.org/stable/book/second-edition/ch17-02-trait-objects.html), _i.e._ to return a `Box<Future>`. This makes the type a little verbose but that should improve very soon!

The actual implementation is a little bit tedious, because we have to parse the
arguments and handle all the cases where they are wrong. To keep
`Service::call()` readable, we implement the request handling on
`CalculatorService` directly, and only use `Service::call()` to dispatch the
request. The code looks like:

```rust
struct CalculatorService;

impl CalculatorService {
    fn handle_request(&self, method: &str, params: &[Value]) -> Result<Value, Value> {
        if params.len() != 2 {
            return Err("Expected two arguments".into());
        }
        if !params[0].is_i64() || !params[1].is_i64() {
            return Err("Invalid argument".into());
        }
        let res = match method {
            "add" => params[0].as_i64().unwrap() + params[1].as_i64().unwrap(),
            "sub" => params[0].as_i64().unwrap() - params[1].as_i64().unwrap(),
            _ => return Err("Unknown method".into()),
        };
        Ok(res.into())
    }
}

impl Service for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = String;
    type Future = Box<Future<Item = Message, Error = String>>;

    fn call(&self, req: Message) -> Self::Future {
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

Notice that the response returned by the service always has its `id` attribute
set to `0`. This is because the protocol handles the IDs for us and the service
does not need to know anything about the protocol.

We can now start serve the service through TCP. We'll use
[`tokio_proto::TcpServer`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/struct.TcpServer.html)
for this:

```rust
fn main() {
    let server = TcpServer::new(Protocol, "127.0.0.1:12345".parse().unwrap());
    server.serve(CalculatorService {});
}
```

The first line creates a `TcpServer` that uses the MessagePack-RPC protocol we
implemented with `tokio-proto`, and that listens on `127.0.0.1:12345`. The
seconds line starts the server, and tells it to run the `CalculatorService`.
Unfortunately:

```
$ cargo run --example  calculator
   Compiling rmp-rpc-demo v0.1.0 (file:///home/little-dude/rust/rmp-rpc-demo)

error[E0277]: the trait bound `CalculatorService: std::ops::Fn<()>` is not satisfied
  --> examples/calculator.rs:53:12
   |
53 |     server.serve(CalculatorService {});
   |            ^^^^^ the trait `std::ops::Fn<()>` is not implemented for `CalculatorService`
   |
   = note: required because of the requirements on the impl of `tokio_service::NewService` for `CalculatorService`

error: aborting due to previous error
error: Could not compile `rmp-rpc-demo`.
```

The
[documentation](https://docs.rs/tokio-service/0.1.0/tokio_service/trait.NewService.html#associatedtype.Error)
shows that `NewService` is a trait that returns a service. This seems a little
bit weird: why would `TcpServer::serve` require a type that _returns_ a service
and not a type that _implements_ a service? The reason for this is that
internally, tokio creates a new service for each client that connects.

Let's listen to the compiler and implement `NewService`. We can do that
directly on `CalculatorService`:

```rust
impl NewService for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = String;
    type Instance = CalculatorService;

    fn new_service(&self) -> io::Result<Self::Instance> {
        Ok(self.clone())
    }
}
```

Aaaand...

```
error[E0277]: the trait bound `std::io::Error: std::convert::From<std::string::String>` is not satisfied
  --> examples/calculator.rs:66:12
   |
66 |     server.serve(CalculatorService {});
   |            ^^^^^ the trait `std::convert::From<std::string::String>` is not implemented for `std::io::Error`
   |
   = help: the following implementations were found:
             <std::io::Error as std::convert::From<rmpv::encode::ValueWriteError>>
             <std::io::Error as std::convert::From<rmp::encode::MarkerWriteError>>
             <std::io::Error as std::convert::From<rmp::encode::DataWriteError>>
             <std::io::Error as std::convert::From<std::ffi::NulError>>
           and 2 others
   = note: required because of the requirements on the impl of `std::convert::Into<std::io::Error>` for `std::string::String`

error: aborting due to previous error

error: Could not compile `rmp-rpc-demo`.
```


The compiler confuses me here: nothing in the `NewService` trait indicates the
trait bound `NewService::Error: Into<io::Error>`. If a reader understands
what's going on here, please let me know. Anywya, since we can't implement
`Into<io::Error>` for `String`, we are left with two options:

- use our own error type that implements `Into<io::Error>`
- use `io::Error` direclty

We'll go for the first one, which is cleaner in my opinion:

```rust
struct ServiceError(String);

impl fmt::Display for ServiceError {
    fn fmt(&self, f: &mut fmt::Formatter) -> Result<(), fmt::Error> {
        write!(f, ServiceError({}), self.0)
    }
}

impl error::Error for DecodeError {
    fn description(&self) -> &str {
        "An error occured while processing a request"
    }

    fn cause(&self) -> Option<&error::Error> { None }
}

impl From<&str> for ServiceError {
    fn from(err: &str) -> Self { ServiceError(err.into()) }
}

impl From<ServiceError> for io::Error {
    fn from(err: ServiceError) -> Self {
        io::Error::new(io::ErrorKind::Other, err.0)
    }
}
```

This is pretty basic. The `From<&str>` implementation allow us to make only
minimal changes to the `Service` and `NewService` traits. All we have to change
is the error type:

```rust
impl NewService for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Instance = CalculatorService;

    // ...
}

impl Service for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Future = Box<Future<Item = Message, Error = ServiceError>>;

    // ...
}
```

`cargo build --example calculator` now works.

The last step is to run the server in background:

```rust

fn main() {
    let addr = "127.0.0.1:12345".parse().unwrap();

    let server = TcpServer::new(Protocol, addr);
    thread::spawn(move || {
		server.serve(CalculatorService {});
    });

    thread::sleep(Duration::from_millis(100));

    // The client code goes here
}
```

## Client implementation

The client is much simpler:

```rust
fn main() {
    let addr = "127.0.0.1:12345".parse().unwrap();

    let server = TcpServer::new(Protocol, addr);
    thread::spawn(move || {
		server.serve(CalculatorService {});
    });

    thread::sleep(Duration::from_millis(100));

    let mut core = Core::new().unwrap();
    let handle = core.handle();

    let connection = TcpClient::new(Protocol).connect(&addr, &handle);
    let requests = connection.and_then(|client| {
        let req = Message::Request(Request {
            method: "add".into(),
            id: 0,
            params: vec![1.into(), 2.into()],
        });
        client.call(req)
            .and_then(move |response| {
                println!("{:?}", response);

                let req = Message::Request(Request {
                    method: "wrong".into(),
                    id: 0,
                    params: vec![],
                });
                client.call(req)
            })
        .and_then(|response| {
            println!("{:?}", response);
            Ok(())
        })
    });
    let _ = core.run(requests);
}
```

This code creates an event loop (`core`), a future (`requests`), and runs this
future until it completes (`core.run()`).  We send two requests, and print the
responses. The first one is a valid requests, and the second one an invalid one
(it has an invalid method name). Just like for the server, we set the ID of the
requests to 0, and rely on the protocol to do the right thing.

When we run it the output should look like:

```
Response(Response { id: 0, result: Ok(Integer(PosInt(3))) })
Response(Response { id: 1, result: Err(String(Utf8String { s: Ok("Unknown method") })) })
```

Some readers may have noticed that we didn't implement the `Service` for the
client, which seems to contradict the diagram I showed in [the chapter
2](introduction/2.md):

![tokio client stack illustration](./images/tokio-stack-client-view.png)

Actually, the `Service` _is_ there. But it is already provided by tokio. The magic operates here:

```rust
let connection = TcpClient::new(Protocol).connect(&addr, &handle);
let requests = connection.and_then(|client| {
    // client implements `Service`! That's why we can do client.call(...)
    // but how come?
});
```

[`tokio_proto::TcpClient::connect()`
returns](https://docs.rs/tokio-proto/0.1.1/tokio_proto/struct.TcpClient.html#method.connect)
a
[`tokio_proto::Connect`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/struct.Connect.html),
which is a future that [returns a
`tokio_proto::BindClient`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/trait.BindClient.html)
when it completes. Here is `BindClient`:
```rust
pub trait BindClient<Kind, T: 'static>: 'static {
    type ServiceRequest;
    type ServiceResponse;
    type ServiceError;
    type BindClient: Service<Request = Self::ServiceRequest, Response = Self::ServiceResponse, Error = Self::ServiceError>;
    fn bind_client(&self, handle: &Handle, io: T) -> Self::BindClient;
}
```

`BindClient` is a complex type, and according to the documentation, it does not
implement `Service`. However, it does have a `BindClient` associated type that
_does_ implement service. It is actually _this_ associated type that is
returned, but we have to [dive in the
sources](https://docs.rs/tokio-proto/0.1.1/src/tokio_proto/tcp_client.rs.html#48)
to see `bind_client` being called.

FIXME: there is still one thing I'm not sure about in the code above. Why does
the `TcpClient::connect()` method takes a handle as argument?  Handles are used
to spawn tasks on the even loop so the only assumption I can make is that
internally, `TcpClient::connect()` spawns a task, but I don't know what.

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
rmpv = "0.4"
tokio-io = "0.1"
tokio-proto = "0.1"

[dev-dependencies]
tokio-core = "0.1"
tokio-service = "0.1"
futures = "0.1"
```

### examples/calculator.rs

```rust
extern crate rmpv;
extern crate rmp_rpc_demo;
extern crate tokio_core;
extern crate tokio_proto;
extern crate tokio_service;
extern crate futures;

use std::{error, io, fmt, thread};
use std::time::Duration;

use tokio_core::reactor::Core;
use tokio_proto::{TcpClient, TcpServer};
use tokio_service::{NewService, Service};
use rmpv::Value;
use futures::{future, Future};
use rmp_rpc_demo::message::{Message, Response, Request};
use rmp_rpc_demo::protocol::Protocol;

#[derive(Clone)]
struct CalculatorService;


impl Service for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Future = Box<Future<Item = Message, Error = ServiceError>>;

    fn call(&self, req: Message) -> Self::Future {
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

impl NewService for CalculatorService {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Instance = CalculatorService;

    fn new_service(&self) -> io::Result<Self::Instance> {
        Ok(self.clone())
    }
}

#[derive(Debug)]
struct ServiceError(String);

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

fn main() {
    let addr = "127.0.0.1:12345".parse().unwrap();

    let server = TcpServer::new(Protocol, addr);
    thread::spawn(move || {
        server.serve(CalculatorService {});
    });

    thread::sleep(Duration::from_millis(100));

    let mut core = Core::new().unwrap();
    let handle = core.handle();

    let connection = TcpClient::new(Protocol).connect(&addr, &handle);
    let requests = connection.and_then(|client| {
        let req = Message::Request(Request {
            method: "add".into(),
            id: 0,
            params: vec![1.into(), 2.into()],
        });
        client.call(req)
            .and_then(move |response| {
                println!("{:?}", response);

                let req = Message::Request(Request {
                    method: "wrong".into(),
                    id: 0,
                    params: vec![],
                });
                client.call(req)
            })
        .and_then(|response| {
            println!("{:?}", response);
            Ok(())
        })
    });
    let _ = core.run(requests);
}
```
