You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-2.3.3)

------------

# Wrapping the client

Just like for the server, there is some boilerplate code that can be spared to
users. This kind of code is un-necessarily verbose for example:

```rust
let req = Message::Request(Request {
    method: "add".into(),
    id: 0,
    params: vec![1.into(), 2.into()],
});
client.call(req).and_then(|response| {
    // ...
});
```

We should not have to build a full `Message` in the first place. To send a
request, users should just have to call a method that takes two arguments: the
method, and the parameters. Let's build a `Client` type with such a method:

```rust
// src/client.rs

use std::io;
use futures::{Future, BoxFuture};
use rmpv::Value;

pub struct Client;

pub type Response = Box<Future<Item = Result<Value, Value>, Error = io::Error>>;

impl Client {
    pub fn request(&self, method: &str, params: Vec<Value>) -> Response {
        // TODO
    }
}
```

This is a bit naive because as it stands, `Client` is an empty type. Instead,
it should wrap a type that implements `Service` and can send requests. It turns
out that tokio provides such a type:
[`tokio_proto::multiplex::ClientService`](https://docs.rs/tokio-proto/0.1.1/tokio_proto/multiplex/struct.ClientService.html).
All we have to do is wrap it and use it to send the requests:

```rust
// src/client.rs

// ...
use tokio_proto::multiplex::ClientService;
use message::{Message, Request};

pub struct Client(ClientService<TcpStream, Protocol>);

pub type Response = Box<Future<Item = Result<Value, Value>, Error = io::Error>>;

impl Client {
    pub fn request(&self, method: &str, params: Vec<Value>) -> Response {
        let req = Message::Request(Request {
            // we can set this to 0 because under the hood it's handle by tokio at the
            // protocol/codec level
            id: 0,
            method: method.to_string(),
            params: params,
        });
        let resp = self.0.call(req).and_then(|resp| {
            match resp {
                Message::Response(response) => Ok(response.result),
                _ => panic!("Response is not a Message::Response");
            }
        });
        Box::new(resp) as Response
    }
}
```

It is easy to adapt the example code to add a `connect()` method as well:

```rust
// src/client.rs

// ...
use tokio_proto::TcpClient;

// ...

impl Client {
    pub fn connect(addr: &SocketAddr, handle: &Handle) -> Box<Future<Item = Client, Error = io::Error>> {
        let ret = TcpClient::new(Protocol)
            .connect(addr, handle)
            .map(Client);
        Box::new(ret)
    }

    // ...
}
```

# Complete code

## client.rs

```rust
use std::io;
use std::net::SocketAddr;

use futures::Future;
use tokio_core::net::TcpStream;
use tokio_core::reactor::Handle;
use tokio_proto::multiplex::ClientService;
use tokio_proto::TcpClient;
use tokio_service::Service;
use rmpv::Value;

use message::{Message, Request};
use protocol::Protocol;


pub struct Client(ClientService<TcpStream, Protocol>);

pub type Response = Box<Future<Item = Result<Value, Value>, Error = io::Error>>;

impl Client {
    pub fn connect(addr: &SocketAddr, handle: &Handle) -> Box<Future<Item = Client, Error = io::Error>> {
        let ret = TcpClient::new(Protocol)
            .connect(addr, handle)
            .map(Client);
        Box::new(ret)
    }

    pub fn request(&self, method: &str, params: Vec<Value>) -> Response {
        let req = Message::Request(Request {
            // we can set this to 0 because under the hood it's handle by tokio at the
            // protocol/codec level
            id: 0,
            method: method.to_string(),
            params: params,
        });
        let resp = self.0.call(req).and_then(|resp| {
            match resp {
                Message::Response(response) => Ok(response.result),
                _ => panic!("Response is not a Message::Response"),
            }
        });
        Box::new(resp) as Response
    }
}
```
