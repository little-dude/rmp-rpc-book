You can find the complete code for this chapter [on github](https://github.com/little-dude/rmp-rpc-demo/tree/chapter-3.1)

------------

# Re-implementing the server

We'll start by implementing the server. So let's empty `src/server.rs`, and
import everything we'll need:

```rust
use std::io;
use std::collections::HashMap;
use std::error::Error;
use std::net::SocketAddr;

use futures::{Async, Poll, Future, Stream, Sink, BoxFuture};
use rmpv::Value;
use tokio_core::net::{TcpStream, TcpListener};
use tokio_core::reactor::Core;
use tokio_io::AsyncRead;
use tokio_io::codec::Framed;

use codec::Codec;
use message::{Response, Message};
```

When working on this library, I kept looking at Tokio's source code. As a
result, I ended up doing things in a very similar way: our library with provide
a `Service` and a `ServiceBuilder` traits, similar to Tokio's `Service` and
`NewService` traits.

Let's start with the `Service` trait, that must be implemented by users who
want build their MessagePack-RPC server. Unlike Tokio's `Service` trait, it
will have two methods instead of one: one to handle requests, and one to handle
notifications.

```rust
pub trait Service {
    type Error: Error;
    type T: Into<Value>;
    type E: Into<Value>;

    fn handle_request(&mut self, method: &str, params: &[Value]) -> BoxFuture<Result<Self::T, Self::E>, Self::Error>;

    fn handle_notification(&mut self, method: &str, params: &[Value]) -> BoxFuture<(), Self::Error>;
}
```

Note that these method do not take a `Message` as argument. This is to avoid
users this kind of code we wrote in the previous example:

```rust
fn call(&self, msg: Message) -> SomeReturnType {
    match msg {
        Message::Request( Request { method, params, .. }) => {
            // handle the request
        }
        Message::Notification ( Notification { method, params, .. }) => {
            // handle the notification
        }
        _ => {
            // handle this as an error
        }
    }
}
```

Also, notice that `Service::handle_request()` returns a future that produces a
`Result<T, E>` where `T` and `E` can be converted into values, instead of a
`Result<Value, Value>`. This will make implementors life a tiny bit easier, and
makes the signature nicer.

For each client, a new `Service` will be created, so we also a builder trait,
similar to `NewService`:

```rust
pub trait ServiceBuilder {
    type Service: Service + 'static;

    fn build(&self) -> Self::Service;
}
```

Finally, for each client a new future will be spawned. This future shall read
the messages coming from the client, dispatch them to the `Service`, and
forward the responses back to the client. This future is not a function but a
type that implements the `Future` trait. We'll call it `Server`. The server
owns a service instance, and a TCP stream connected to the client.

```rust
struct Server<S: Service> {
    service: S,
    stream: TcpStream
    // ...
}

impl<S> Server<S> where S: Service + 'static {
    fn new(service: S, tcp_stream: TcpStream) -> Self {
        Server { service: service, stream: tcp_stream }
    }
}

impl<S> Future for Server<S> where S: Service + 'static {
    type Item = ();
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        // TODO:
        //  - read incoming messages from the client
        //  - dispatch them to the Service
        //  - send the responses back to the client
    }
}
```

We'll start by turning the `TcpStream` into `Framed<TcpStream, Codec>` where
`Codec` is the MessagePack-RPC codec we wrote earlier. That will allow us to
read and write `Message` instead of bytes:

```
struct Server<S: Service> {
    service: S,
    stream: Framed<TcpStream, Codec>
    // ...
}

impl<S> Server<S> where S: Service + 'static {
    fn new(service: S, tcp_stream: TcpStream) -> Self {
        Server { service: service, stream: tcp_stream.framed(Codec) }
    }
}
```

The biggest task is to implement `Future`. The first thing it has to do is to
read the incoming messages and dispatch them to the service:

```rust

impl<S> Future for Server<S> where S: Service + 'static {
    type Item = ();
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        // Read the messages from the client
        loop {
            // Poll the stream
            match self.stream.poll().unwrap() {
                // We got a message, let's dispatch it to the Service
                Async::Ready(Some(msg)) => self.dispatch_message(msg),
                // The stream finished, probably because the client closed the connection.
                // There is no need for the server to continue so we return Async::Ready(())
                // to notify tokio that this future finished
                Async::Ready(None) => return Ok(Async::Ready(())),
                // There is no new message ready to be read.
                // Let's just break out of this loop
                Async::NotReady => break,
            }
        }

        // TODO: send back the responses that the service procuced, if any

        // Important! Notify tokio that this future is not ready yet. By
        // returning Async::NotReady we tell tokio to re-schedule this future for
        // polling.
        Ok(Async::NotReady)
    }
}
```

It's important to understand that when spawning a `Server` instance on a tokio
event loop, tokio will keep calling `poll()` as long as it return
`Ok(Async::NotReady)`. In other words, the server runs until `poll()` returns
an error or `Ok(Async::Ready(()))`.

Let's take a closer look at the `Server::dispatch_message()` method:

```rust
impl<S> Server<S> where S: Service + 'static {

    // ...

    fn dispatch_message(&mut self, msg: Message) {
        match msg {
            Message::Request(request) => {
                let method = request.method.as_str();
                let params = request.params;
                let response = self.service.handle_request(method, &params);
            }
            Message::Notification(notification) => {
                let method = notification.method.as_str();
                let params = notification.params;
                let outcome = self.service.handle_notification(method, &params);
            }
            // Let's just silently ignore responses.
            Message::Response(_) => return;
        }
    }
}
```

Pretty straightforward except that... we need to keep track of the futures
returned by `handle_request()` and `handle_notfication()` of course. Since
responses have an ID, it seems natural to use a hashmap and use the ID as a
key. For the notifications, a vector is enough:


```rust
struct Server<S: Service> {
    service: S,
    stream: Framed<TcpStream, Codec>,
    request_tasks: HashMap<u32, BoxFuture<Result<S::T, S::E>, S::Error>>,
    notification_tasks: Vec<BoxFuture<(), S::Error>>,
}

impl<S> Server<S> where S: Service + 'static {
    fn new(service: S, tcp_stream: TcpStream) -> Self {
        Server {
            service: service,
            stream: tcp_stream.framed(Codec),
            request_tasks: HashMap::new(),
            notification_tasks: Vec::new(),
        }
    }

    fn dispatch_message(&mut self, msg: Message) {
        match msg {
            Message::Request(request) => {
                let method = request.method.as_str();
                let params = request.params;
                let response = self.service.handle_request(method, &params);
                // store the future, so that we can poll it until it completes
                self.request_tasks.insert(request.id, response);
            }
            Message::Notification(notification) => {
                let method = notification.method.as_str();
                let params = notification.params;
                let outcome = self.service.handle_notification(method, &params);
                // store the future, so that we can poll it until it completes
                self.notification_tasks.push(outcome);
            }
            Message::Response(_) => return,
        }
    }
}
```

It remains to poll the futures returned by our service until they complete.
Futures returned by `handle_request()` return a `Response` that needs to be
sent back the client. Futures returned by `handle_notification()` don't return
anything, but it's important to poll them as well, to run them to completion.

We'll create two separate methods to poll these futures:
`Server::process_requests()` and `Server::process_notifications()`:

```rust

impl<S> Server<S> where S: Service + 'static {

    // ...

    fn process_notifications(&mut self) {
        // keep track of the futures that finished
        let mut done = vec![];

        for (idx, task) in self.notification_tasks.iter_mut().enumerate() {
            match task.poll().unwrap() {
                // the future finished
                Async::Ready(_) => done.push(idx),
                // the future did not finished yet
                Async::NotReady => continue,
            }
        }

        // stop tracking the futures that finished
        for idx in done.iter().rev() { self.notification_tasks.remove(*idx); }
    }

    fn process_requests(&mut self) {
        // keep track of the futures that finished
        let mut done = vec![];

        for (id, task) in &mut self.request_tasks {
            match task.poll().unwrap() {
                // this future finished. we send the response back to the client
                Async::Ready(response) => {
                    let msg = Message::Response(Response {
                        id: *id,
                        result: response.map(|v| v.into()).map_err(|e| e.into()),
                    });
                    done.push(*id);

                    // send the response back to the client
                    if !self.stream.start_send(msg).unwrap().is_ready() {
                        panic!("the sink is full")
                    }
                }
                // the future did not finished yet. We ignore it.
                Async::NotReady => continue,
            }
        }

        // stop tracking the futures that finished
        for idx in done.iter_mut().rev() {
            let _ = self.request_tasks.remove(idx);
        }
    }
```

I think the comments make the code clear enough. Of course we need to call these methods in `poll()`:

```rust

impl<S> Future for Server<S> where S: Service + 'static {
    type Item = ();
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        loop {
            match self.stream.poll().unwrap() {
                Async::Ready(Some(msg)) => self.handle_msg(msg),
                Async::Ready(None) => {
                    return Ok(Async::Ready(()));
                }
                Async::NotReady => break,
            }
        }
        self.process_notifications();
        self.process_requests();
        Ok(Async::NotReady)
    }
}
```

We are almost done, but one subtility remains. If you noticed, when sending the
response, we use `stream.start_send()`. The
[documentation](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html#tymethod.start_send)
explains that this is not enough to actually send the response. Internally, for
performance reasons, the `Sink` may buffer the messages before sending them.
With only `start_send()` we cannot be sure that the data wether has actually
being written into the TCP socket, or wether the operation is pending.

To make sure the data is sent, we need to tell the sink to flush its buffers
with
[`Sink::poll_complete()`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html#tymethod.poll_complete).
We could do this for each message, right after `start_send()`, but this would
kind of defeat the purpose of buffering.  Instead, we'll flush after we
processed all the requests:


```rust
impl<S> Future for Server<S> where S: Service + 'static {
    type Item = ();
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        loop {
            match self.stream.poll().unwrap() {
                Async::Ready(Some(msg)) => self.handle_msg(msg),
                Async::Ready(None) => {
                    return Ok(Async::Ready(()));
                }
                Async::NotReady => break,
            }
        }
        self.process_notifications();
        self.process_requests();
        self.stream.poll_complete().unwrap();
        Ok(Async::NotReady)
    }
}
```

Finally, we could try to make users life easier by providing a function to run
a server on a Tokio event loop. It's not mandatory, but it will also show how
the code we just wrote can be used:

```rust
pub fn serve<B: ServiceBuilder>(address: &SocketAddr, service_builder: B) {
    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let listener = TcpListener::bind(address, &handle).unwrap();
    core.run(listener.incoming().for_each(|(stream, _address)| {
        let service = service_builder.build();
        let server = Server::new(service, stream);
        handle.spawn(server.map_err(|_| ()));
        Ok(())
    })).unwrap()
}
```

A TCP listener creates a new service for each client that connects, and spawn a
new `Server` instance on the event loop. Once the server is spawned, Tokio
start's calling `poll()` and the server. This is kind of confusing, because our
MessagePack-RPC server actually works by spawning multiple `Server` (one per
client).
