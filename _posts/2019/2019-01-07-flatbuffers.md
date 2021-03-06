---
layout: post
title: FlatBuffers (in Rust)
tags:
- rust
- flatbuffers
- showdev
---

Following up on the [last post closing the chapter on Apache Thrift]({% post_url /2019/2019-01-06-thrift_012 %}), we're looking at another serialization library, [FlatBuffers](https://google.github.io/flatbuffers/).

## FlatBuffers

A lesson learned from using Thrift was we wanted performant, schema'd serialization and robust messaging, but _not_ tightly coupled together.  We ended up using the message framing and other features, but [using ZeroMQ as a transport]({% post_url /2018/2018-09-07-zeromq-thrift %}) and implementing message identification for pub/sub (as opposed to Thrift's request/reply model).

Other strikes against Thrift we didn't discover until later.  The C++ client has a [boost](https://www.boost.org/) dependency and requires both exceptions and [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information).  Boost you either love or hate, but the last two are basically verboten in the video game industry.  Case in point, getting our [SDK working](https://github.com/subor/sample_ue4_platformer) in [UE4](https://www.unrealengine.com/) was [a hassle](https://github.com/subor/sdk/blob/master/docs/topics/ue4.md).

Potentially better performance and reduced memory churn drove us to evaluate options like [Cap'n Proto](https://capnproto.org/) and we're leaning towards [FlatBuffers](https://google.github.io/flatbuffers/).  Notes:

- FlatBuffers is backed by Google
- Cap'n [C# support is dubious](https://capnproto.org/otherlang.html) at best
- Cap'n likely has an edge in performance ([[0]](https://github.com/thekvs/cpp-serializers), [[1]](https://github.com/felixguendling/cpp-serialization-benchmark))
- Both have first-class support for [Rust](https://www.rust-lang.org/) ([FlatBuffers](https://github.com/google/flatbuffers#supported-programming-languages), [Cap'n Proto](https://github.com/capnproto/capnproto-rust))
- Unity support: FlatBuffers' C# library [targets .NET 3.5](https://github.com/google/flatbuffers/blob/master/net/FlatBuffers/FlatBuffers.csproj#L12) and ["just works"](http://exiin.com/blog/flatbuffers-for-unity-sample-code/).  There's a [dormant github project](https://github.com/ThomasBrixLarsen/capnproto-dotnet) for Cap'n.
- FlatBuffers has better adoption in video game industry ([google](https://developers.google.com/games/#Tools), [cocos2d-x](https://cocos2d-x.org/reference/native-cpp/V3.5/d7/d2d/namespaceflatbuffers.html))

The last two are significant deciding factors for us.  There's a better chance a developer is already using FlatBuffers and that's one less dependency our SDK will introduce.

## FlatBuffers in Rust

[Documentation](https://docs.rs/flatbuffers/0.5.0/flatbuffers/) is a bit light, although they have a ["Use in Rust" guide](https://google.github.io/flatbuffers/flatbuffers_guide_use_rust.html) in the FlatBuffers documenation.

Instead of following the directions (I know, I know) and using `HEAD`, we're using the [latest stable release, 1.10.0](https://github.com/google/flatbuffers/releases).

FlatBuffers schema `protos/bench.fbs`:
```thrift
namespace bench;

table Basic {
  id:ulong;
}

table Complex {
  name:string (required);
  basic:bench.Basic (required);
  reference:string (required);
}
```
Run `flatc --rust -o src protos/bench.fbs` to generate Rust source `src/bench_generated.rs` containing two structs: `bench::Basic` and `bench::Complex`.

We can then serialize/deserialize `bench::Basic` struct:
```rust
use crate::bench_generated as bench_fbs;

#[test]
fn it_deserializes() {
    const ID: u64 = 12;
    let mut builder = FlatBufferBuilder::new();
    // bench::Basic values that will be serialized
    let basic_args = bench_fbs::bench::BasicArgs { id: ID, .. Default::default() };
    // Serialize bench::Basic struct
    let basic: WIPOffset<_> = bench_fbs::bench::Basic::create(&mut builder, &basic_args);
    // Must "finish" the builder before calling `finished_data()`
    builder.finish_minimal(basic);
    // Deserialize the bench::Basic
    let parsed = flatbuffers::get_root::<bench_fbs::bench::Basic>(builder.finished_data());
    assert_eq!(parsed.id(), ID);
}
```
Notes:
- `.. Default::default()` isn't needed here, but shows [`impl Default`](https://doc.rust-lang.org/std/default/trait.Default.html) is also generated
- `create()` serializes the struct and returns a `WIPOffset<bench::Basic>` which is an offset to the root of the data
- The documents deserialize with `get_root_as_XXX()` functions which aren't generated by flatc 1.10 (need HEAD?) [but appear to be](https://github.com/google/flatbuffers/issues/5000) wrappers around `get_root()`.

For the `bench::Complex` struct:
```rust
use crate::bench_generated as bench_fbs;

#[test]
fn it_deserializes() {
    const ID: u64 = 12;
    const NAME: &str = "name";
    const REFERENCE: &str = "reference";
    let mut builder = flatbuffers::FlatBufferBuilder::new();
    {
        let args = bench_fbs::bench::BasicArgs{id: ID};
        let basic = Some(bench_fbs::bench::Basic::create(&mut builder, &args));
        let name = Some(builder.create_string(NAME));
        let reference = Some(builder.create_string(REFERENCE));
        let args = bench_fbs::bench::ComplexArgs{ basic, name, reference };
        let complex = bench_fbs::bench::Complex::create(&mut builder, &args);
        builder.finish_minimal(complex);
    }
    let parsed = flatbuffers::get_root::<bench_fbs::bench::Complex>(builder.finished_data());
    assert_eq!(parsed.basic().id(), ID);
    assert_eq!(parsed.name(), NAME);
    assert_eq!(parsed.reference(), REFERENCE);
}
```

Notes:
- Needing to manually serialize each member of `bench::Complex` is cumbersome and error-prone.  There doesn't seem to be a way to automatically handle it...

## Benchmarks

The above schema is from the [proto_benchmarks](https://github.com/ChrisMacNaughton/proto_benchmarks) repository which compares protobuf with capnproto.  I stumbled upon it and swooned over the pretty [criterion](https://github.com/bheisler/criterion.rs) plots.  [I forked it](https://github.com/jeikabu/proto_benchmarks) to add FlatBuffers and migrate to [Rust 2018](https://blog.rust-lang.org/2018/07/27/what-is-rust-2018.html).

The flatbuffers schema is actually auto-magically generated from the protobuf schema:
```protobuf
syntax = "proto2";
option optimize_for = SPEED;

package bench;

message Basic {
    required uint64 id = 1;
}

message Complex {
    required string name = 1;
    required Basic basic = 2;
    required string reference = 3;
}
```

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

Using [`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) to execute `flatc`.  First, to convert the protobuf schema into FlatBuffers, then to output Rust source.

### Results

Not entirely sure I believe the results.  Not knowing much about any of the three libraries, we're likely comparing slightly different things.

Basic write:  
![]({{ "/assets/flatbuffers_rust_basic_write.png" | absolute_url }})

Complex build:  
![]({{ "/assets/flatbuffers_rust_complex_build.png" | absolute_url }})

Also found [rust-serialization-benchmarks](https://github.com/erickt/rust-serialization-benchmarks) which hasn't been updated in 3 years and seems to use [Bencher](https://doc.rust-lang.org/1.1.0/test/struct.Bencher.html) for benchmarking.
