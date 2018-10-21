---
layout: post
title: ASP.NET Core Generic Host with Rust Services
tags:
- rust
- nng
- csharp
- netcore
---

[Last time]({% post_url /2018/2018-09-30-rust-ffi-ci %}) 
Now I'm going to load the Rust binary as part of a NetCore application and get C# and Rust communicating with Nng and exchange a Thrift message.



## Specialization

https://play.rust-lang.org/?version=nightly&mode=debug&edition=2015
https://github.com/rust-lang/rust/issues/31844
https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md
https://stackoverflow.com/questions/34471212/how-to-implement-specialized-versions-of-a-generic-function

`Send` for pointer types
https://github.com/rust-lang/rust/issues/21709

## C# Interop

https://dev.to/living_syn/calling-rust-from-c-6hk

```
cargo new --lib --name rust_input
```

Defaults to producing static libraries, to generate a dynamic/shared library to `Cargo.toml` add:
```toml
[lib]
crate-type = ["dylib"]
```

This will produce a `.dylib` on OSX, `.so` on Linux, `.dll` on Windows, etc.  Also see the [cargo docs](https://doc.rust-lang.org/cargo/reference/manifest.html#building-dynamic-or-static-libraries).

In `lib.rs`:
```rust
#[no_mangle]
pub extern fn start() -> i32 {
    println!("Start!");
    0
}
```

Run `cargo build`.

For immediate satisfaction I copied it to my .net output folder, but I'll need to look into [loading it as a native assembly]({% post_url /2018/2018-09-09-native-assembly %}).

In the C#:
```csharp
[DllImport("rust_input")]
static extern int start();

protected override Task ExecuteAsync(CancellationToken token)
{
    return Task.Run(async () => {
        // Arbitrary sleep to give the broker time to start
        await Task.Delay(1000);
        start();
        while (!token.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromMilliseconds(200));
        }
    });
}
```

## Json Config

[Serde](https://serde.rs/) specifically [Json](https://github.com/serde-rs/json)

Given the following `appsettings.json`:
```json
{
    "zxy":{
        "http":{
            "port":8283
        },
        "nng":{
            "brokerIn": "tcp://localhost:10110",
            "brokerOut": "tcp://localhost:10111"
        }
    },
    "urls": "http://*:8284"
}
```

In `Cargo.toml`:
```toml
[dependencies]
serde = "1.0.79"
serde_json = "1.0.31"
serde_derive = "1.0.79"
```

```rust
extern crate serde;
extern crate serde_json;
// Import macros from serde_derive.  Must appear before first use.
#[macro_use]
extern crate serde_derive;

use std::fs::File;

#[derive(Deserialize,Debug)]
struct AppSettings {
    zxy: ZxySettings
}

#[derive(Deserialize,Debug)]
struct ZxySettings {
    http: HttpSettings,
    nng: NngSettings,
}

#[derive(Deserialize,Debug)]
struct HttpSettings {
    port: u16
}

#[derive(Deserialize,Debug)]
struct NngSettings {
    brokerIn: String,
    brokerOut: String,
}

fn load_settings() -> AppSettings {
    let file = File::open("appsettings.json").unwrap();
    // Deserialize AppSettings from file
    let settings: AppSettings = serde_json::from_reader(file).unwrap();
    println!("{:?}", settings);
    settings
}
```

Importance of location of `#[macro_use]` comes from [this SO](https://stackoverflow.com/questions/29068716/how-do-you-use-a-macro-from-inside-its-own-crate).

## NNG

Use the [runng crate](https://crates.io/crates/runng) we [developed previously to expose NNG to Rust]({% post_url /2018/2018-10-17-rust-ffi-nng %}):
```rust
extern crate runng;
extern crate futures;
use runng::{Factory, Dial};
use runng::protocol::{AsyncPublish, AsyncSocket};
use runng::msg::NngMsg;
use futures::future::Future;

#[no_mangle]
pub extern fn start() -> i32 {
    println!("Start!");

    let setting = load_settings();

    let factory = runng::Latest::new();
    let pusher = factory.pusher_open().unwrap();
    println!("Connecting....");
    pusher.dial(&setting.zxy.nng.brokerIn).unwrap();
    println!("Connected!");
    let mut pusher = pusher.create_async_context().unwrap();
    let mut msg = NngMsg::new().unwrap();
    msg.append_u32(0).unwrap(); // For topic appends 4 bytes: 0 0 0 0
    println!("Sending...");
    pusher.send(msg).wait().unwrap().unwrap();
    println!("Sent!");

    0
}
```

We can subscribe to this in C#:
```csharp
// Load configuration and NNG native dll
var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();
var brokerOut = config.GetSection("zxy:nng").GetValue<string>("brokerOut");
var factory = LoadNngFactory();

// Create subscriber that connects to output/publishing end of broker
using (var subscriber = factory.SubscriberCreate(brokerOut).Unwrap().CreateAsyncContext(factory).Unwrap())
{
    var topic = new byte[]{0, 0, 0, 0};
    subscriber.Subscribe(topic);
    Console.WriteLine("Receiving...");
    var msg = await subscriber.Receive(cts.Token);
    Console.WriteLine("Received!");
}
```

One thing that's fantastic about our microservice architecture is once we start a Rust service and it connects to NNG, we can interact with it the same as our C# services.  No need to deal with managed to unmanaged interop which gets pretty hairy for non-trivial types.

## Thrift

Which is ok, but it's going to get tedious (and error-prone) to maintain structs here (as well as in C#).
