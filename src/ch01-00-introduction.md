# Introduction

The first part of this book does not contain any code. It explains some basics
about the library we'll build, and the tools we'll use.

- [chapter 1.1](ch01-01-messagepack-rpc.md) explains what MessagePack-RPC is and what kind of problems is solves
- [chapter 1.2](ch01-02-tokio.md) gives an overview of the Tokio stack

To make it easier to follow and to skip chapters, all the code is also
available on github, in the [rmp-rpc-demo repository](https://github.com/little-dude/rmp-rpc-demo).
There is a branch per chapter.

The content of this book is also available on
[github](https://github.com/little-dude/rmp-rpc-book), so feel free to submit
PRs or file issues.


## Why this book?

This guide does not intend to be a substitute to the [tokio documentation](https://tokio.rs/docs/getting-started/tokio/).

Actually, if you're starting with Tokio, you will probably have to look into
the official documentation quite often. The intention is to provide a concrete
example of how to build things with Tokio. The documentation does a great jobs
at explaining the concepts behind Tokio and introducing the different pieces of
the ecosystem. But there are so many concepts to understand, and so many crates
available that it can be hard at the beginning to know what you really need to
build something (at least, that was my feeling when I started working on my
[first crate](https://github.com/little-dude/rmp-rpc). By building a real world
protocol step by step, we aim at gradually putting into application the
concepts and crates presented in the official documentation.


## Who is the target audience?

People who already know the basic concepts of Rust, and want to get started
with Tokio.

## Disclaimer

Two things:

- I am pretty much a beginner with Tokio and this book has not been under any
  review by qualified people yet. It may promote anti-patterns, contain errors,
  imprecisions, etc.
- This is work in progress. There are _many_ things I'd like to add: more
  explanations, more detailed examples, TLS support, graceful shutdown for the
  server, etc. etc.
