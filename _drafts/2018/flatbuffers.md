---
layout: post
title: FlatBuffers
tags:
- rust
- flatbuffers
- showdev
---

Since using [protobuf](https://github.com/protocolbuffers/protobuf) on a former project I've developed a love (obsession?) for serialization libraries.  On our current project we use [Apache Thrift](https://github.com/apache/thrift) and on an a new project we're going ahead with [FlatBuffers](https://google.github.io/flatbuffers/).


## FlatBuffers

A lesson learned from using Thrift was we wanted robust messaging, but _not_ tightly coupled to our message serialization.  We ended up using its message framing and other features, but [using ZeroMQ as a transport]({% post_url /2018/2018-09-07-zeromq-thrift %}) and implementing message identification for pub/sub (as opposed to Thrift's request/reply).

Other strikes against Thrift we didn't discover until later.  The C++ client has a [boost](https://www.boost.org/) dependency and requires both exceptions and [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information).  Boost you either love or hate, but the last two are basically verboten in the video game industry.  Case in point, getting our [SDK working](https://github.com/subor/sample_ue4_platformer) in [UE4](https://www.unrealengine.com/) was [a hassle](https://github.com/subor/sdk/blob/master/docs/topics/ue4.md).

Potentially better performance and reduced memory churn drove us to evaluate [Cap'n Proto](https://capnproto.org/) and ultimately settle on [FlatBuffers](https://google.github.io/flatbuffers/).  Notes:

- FlatBuffers is backed by Google
- Both have first-class support for [Rust](https://www.rust-lang.org/) ([FlatBuffers](https://github.com/google/flatbuffers#supported-programming-languages), [Cap'n Proto](https://github.com/capnproto/capnproto-rust))
- Cap'n's [C# support is dubious](https://capnproto.org/otherlang.html) at best
- Cap'n likely has an edge in performance ([0](https://github.com/thekvs/cpp-serializers), [1](https://github.com/felixguendling/cpp-serialization-benchmark))
- Unity support: FlatBuffers' C# library [targets .NET 3.5](https://github.com/google/flatbuffers/blob/master/net/FlatBuffers/FlatBuffers.csproj#L12) and ["just works"](http://exiin.com/blog/flatbuffers-for-unity-sample-code/).  There's a [dormant github project](https://github.com/ThomasBrixLarsen/capnproto-dotnet) for Cap'n.
- FlatBuffers has better adoption in video game industry ([google](https://developers.google.com/games/#Tools), [cocos2d-x](https://cocos2d-x.org/reference/native-cpp/V3.5/d7/d2d/namespaceflatbuffers.html))

The last two were the largest deciding factors.  There's a better chance a developer is already using FlatBuffers and that's one less dependency our SDK will introduce.

## Rust

[Documentation](https://docs.rs/flatbuffers/0.5.0/flatbuffers/) is a bit light, they have a ["Use in Rust" guide](https://google.github.io/flatbuffers/flatbuffers_guide_use_rust.html) in the FlatBuffers documenation.

Instead of following the directions I'm using the [latest stable release, 1.10.0](https://github.com/google/flatbuffers/releases).

https://github.com/ChrisMacNaughton/proto_benchmarks

In `build.rs`:
```rust
// Convert protobuf .proto to FlatBuffers .fbs
std::process::Command::new("flatc")
    .args(&["--proto", "-o", "protos", "protos/bench.proto"])
    .spawn()
    .expect("flatc");
// Generate rust source
std::process::Command::new("flatc")
    .args(&["--rust", "-o", "src", "protos/bench.fbs"])
    .spawn()
    .expect("flatc");
```

Using [`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) to execute `flatc` command-line program to produce `src/bench_generated.rs`.

We can then serialize/deserialize `bench::Basic`:
```rust
#[test]
fn it_deserializes() {
    const id: u64 = 12;
    let mut builder = FlatBufferBuilder::new();
    let basic_args = bench_generated::bench::BasicArgs { id };
    let basic = bench_generated::bench::Basic::create(&mut builder, &basic_args);
    builder.finish_minimal(basic);
    let parsed = flatbuffers::get_root::<bench_generated::bench::Basic>(builder.finished_data());
    assert_eq!(parsed.id(), id);
}
```

## Benchmarks

https://github.com/erickt/rust-serialization-benchmarks