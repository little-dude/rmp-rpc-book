# Complete code

## server.rs


```rust
use std::io;
use std::collections::HashMap;
use std::error::Error;
use std::net::SocketAddr;

use codec::Codec;
use message::{Response, Message};

use futures::{Async, Poll, Future, Stream, Sink, BoxFuture};
use rmpv::Value;
use tokio_core::net::{TcpStream, TcpListener};
use tokio_core::reactor::Core;
use tokio_io::AsyncRead;
use tokio_io::codec::Framed;

pub trait Service {
    type Error: Error;
    type T: Into<Value>;
    type E: Into<Value>;

    fn handle_request(
        &mut self,
        method: &str,
        params: &[Value],
    ) -> BoxFuture<Result<Self::T, Self::E>, Self::Error>;

    fn handle_notification(&mut self, method: &str, params: &[Value]) -> BoxFuture<(), Self::Error>;
}

pub trait ServiceBuilder {
    type Service: Service + 'static;

    fn build(&self) -> Self::Service;
}

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
                self.request_tasks.insert(request.id, response);
            }
            Message::Notification(notification) => {
                let method = notification.method.as_str();
                let params = notification.params;
                let outcome = self.service.handle_notification(method, &params);
                self.notification_tasks.push(outcome);
            }
            Message::Response(_) => {
                return;
            }
        }
    }

    fn process_notifications(&mut self) {
        let mut done = vec![];
        for (idx, task) in self.notification_tasks.iter_mut().enumerate() {
            match task.poll().unwrap() {
                Async::Ready(_) => done.push(idx),
                Async::NotReady => continue,
            }
        }
        for idx in done.iter().rev() {
            self.notification_tasks.remove(*idx);
        }
    }

    fn process_requests(&mut self) {
        let mut done = vec![];
        for (id, task) in &mut self.request_tasks {
            match task.poll().unwrap() {
                Async::Ready(response) => {
                    let msg = Message::Response(Response {
                        id: *id,
                        result: response.map(|v| v.into()).map_err(|e| e.into()),
                    });
                    done.push(*id);
                    if !self.stream.start_send(msg).unwrap().is_ready() {
                        panic!("the sink is full")
                    }
                }
                Async::NotReady => continue,
            }
        }

        for idx in done.iter_mut().rev() {
            let _ = self.request_tasks.remove(idx);
        }
    }
}

impl<S> Future for Server<S> where S: Service + 'static {
    type Item = ();
    type Error = io::Error;

    fn poll(&mut self) -> Poll<Self::Item, Self::Error> {
        loop {
            match self.stream.poll().unwrap() {
                Async::Ready(Some(msg)) => self.dispatch_message(msg),
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
