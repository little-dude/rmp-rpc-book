# First implementation

In this part, we'll implement an incomplete MessagePack-RPC library using the
whole tokio stack:

- `tokio-io` for the codec (see [chapter 2.1](ch02-01-00-codec.md))
- `tokio-proto` for the protocol (see [chapter 2.2](ch02-02-00-protocol.md))
- `tokio-service` for the service (see [chapter 2.3](ch02-03-00-service.md))

Why an "incomplete" library? We'll see that using only `tokio-proto`, we cannot
implement support for notifications. But worry not! this is something we'll fix
in [part 3](ch03-00-second-implementation).
