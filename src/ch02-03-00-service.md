# Service

The last piece of the Tokio stack is the `Service` trait. Our library does not
have to provide a `Service` implementation. Users can already build a
MessagePack-RPC server and client with the `Protocol` implementation the
library provides. In this chapter we'll build a simple example server using the
library as it stands. Then, we'll show how to provide a custom `Service`
implementation to make the library more user friendly.
