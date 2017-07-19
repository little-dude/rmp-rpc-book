You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-2.3.4)

------------

# Updated example


TODO: step by step explanations


## Complete code

```rust
extern crate rmpv;
extern crate rmp_rpc;
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
use rmp_rpc::message::{Message, Response, Request};
use rmp_rpc::protocol::Protocol;

#[derive(Clone)]
struct CalculatorServer;

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

impl NewService for CalculatorServer {
    type Request = Message;
    type Response = Message;
    type Error = ServiceError;
    type Instance = CalculatorServer;

    fn new_service(&self) -> io::Result<Self::Instance> {
        Ok(self.clone())
    }
}

impl Service for CalculatorServer {
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
        server.serve(CalculatorServer {});
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
