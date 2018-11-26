---
layout: post
title: ASP.NET Core Generic Host with Rust Services
tags:
- rust
- csharp
- netcore
- showdev
- nng
- thrift
---

This has been a long time coming, previously we:
- [Thrift client in Rust]({% post_url /2018/2018-08-30-rust-thrift %})
- [Rust FFI bindings to NNG]({% post_url /2018/2018-09-30-rust-ffi-ci %})
- [ASP.NET Core generic host application with NNG]({% post_url /2018/2018-10-21-aspnetcore-uservice-nng %})

Now we're going to load a Rust binary as part of our .NET Core application and get C# and Rust communicating with NNG and Thrift.

## C# Interop

Came across [this highly relevant blog](https://dev.to/living_syn/calling-rust-from-c-6hk).  Start a new Rust library:
```
cargo new --lib --name rust_input
```

Which defaults to producing static libraries.  In order to generate a dynamic/shared library, to `Cargo.toml` add:
```toml
[lib]
crate-type = ["dylib"]
```

This will produce a `.dylib` on OSX (and presumably a `.so` on Linux and `.dll` on Windows).  Also see the [cargo docs](https://doc.rust-lang.org/cargo/reference/manifest.html#building-dynamic-or-static-libraries).

__Update 2018/11/26__

The [2018 edition guide mentions `cdylib`](https://rust-lang-nursery.github.io/edition-guide/rust-2018/platform-and-target-support/cdylib-crates-for-c-interoperability.html) crate type.  In release, it results in a 780508 byte dynamic library instead of 1334912 bytes.

In `lib.rs`:
```rust
#[no_mangle]
pub extern fn start() -> i32 {
    println!("Start!");
    0
}
```

Run `cargo build`.

For immediate satisfaction, we can copy the generated `target/debug/librust_input.dylib` to our .NET output folder, but we'll need to look into [loading it as a native assembly]({% post_url /2018/2018-09-09-native-assembly %}).

In C# we'll create [a background service]({% post_url /2018/2018-10-21-aspnetcore-uservice-nng %}):
```csharp
public class InputXy : BackgroundService
{
    [DllImport("rust_input")]
    static extern int start();

    protected override Task ExecuteAsync(CancellationToken token)
    {
        return Task.Run(async () => {
            // Arbitrary sleep to give the broker time to start
            await Task.Delay(500);
            // Call `start()` in "rust_input" shared library
            start();
            while (!token.IsCancellationRequested)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(200));
            }
        });
    }
}
```

Note the lack of file-extension (or `lib` prefix) with `DllImport`.  This allows the correct library to be found on any platform (i.e. OSX, Linux, Windows).

This works, but doesn't yet do anything interesting.

## Configuration

Our .Net application [uses configuration from `appsettings.json`]({% post_url /2018/2018-10-21-aspnetcore-uservice-nng %}#configuration), and we'd like to use the same settings in Rust.

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
}
```

To deserialize this we can use [Serde](https://serde.rs/), specifically [serde_json](https://github.com/serde-rs/json).

In `Cargo.toml`:
```toml
[dependencies]
serde = "1.0.79"
serde_json = "1.0.31"
serde_derive = "1.0.79"
```

In `lib.rs`:
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

This produces `AppSettings` structure containing values from `appsettings.json`.  Now we can easily use the same configuration in both C# and Rust.

## NNG

Use our [runng crate](https://crates.io/crates/runng) [from before]({% post_url /2018/2018-10-17-rust-ffi-nng %}) to create a ["push"](https://nanomsg.github.io/nng/man/v1.0.0/nng_push.7.html) node connected to our C# broker:
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

We can subscribe to this from C#:
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
    // Use 0x00000000 (four bytes) for our topic
    var topic = new byte[]{0, 0, 0, 0};
    subscriber.Subscribe(topic);
    Console.WriteLine("Receiving...");
    var msg = await subscriber.Receive(cts.Token);
    Console.WriteLine("Received!");
}
```

## Scenic Route

And now we get to the part that delayed this post.

Similar to how we structure [our SDK](https://subor.github.io/api/cs/en-US/html/87e8780b-8e28-7c0d-fa86-89f98b06162b.htm), we want all the Thrift interfaces in a central library we reference from our various services.

`cargo new --lib --name zxy` and in `Cargo.toml`:
```toml
[package]
name = "zxy"
version = "0.1.0"

[dependencies]
thrift = "0.0.4"
try_from = "0.2"
ordered-float = "1.0"
```

`lib.rs`:
```rust
extern crate ordered_float;
extern crate try_from;
extern crate thrift;
```

Back in `rust_input/Cargo.toml`:
```
[dependencies]
runng = { version = "0.1.1", path = "../../../../rust/runng/runng" }
zxy = { path = "../../zxy" }
```

Run and... __fail__:
```
Exception has occurred: CLR/System.DllNotFoundException
Exception thrown: 'System.DllNotFoundException' in input.dll: 'Unable to load shared library 'rust_input' or one of its dependencies. In order to help diagnose loading problems, consider setting the DYLD_PRINT_LIBRARIES environment variable: dlopen(librust_input, 1): image not found'
```

With `DYLD_PRINT_LIBRARIES` enabled:
```
dyld: loaded: /XXX/zxy/output/Debug/plugins/netstandard2.0/librust_input.dylib
dyld: unloaded: /XXX/zxy/output/Debug/plugins/netstandard2.0/librust_input.dylib
```

Not particularly helpful.

On OSX use `otool` to check shared library dependencies (on Windows we usually use [Depedency Walker](http://dependencywalker.com/)):
```bash
$ otool -L target/debug/librust_input.dylib
target/debug/librust_input.dylib:
        /XXX/zxy/target/debug/deps/librust_input.dylib (compatibility version 0.0.0, current version 0.0.0)
        @rpath/libstd-ffe37452bb8eb44d.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
        /usr/lib/libresolv.9.dylib (compatibility version 1.0.0, current version 1.0.0)
```

Now remove `zxy` from `[dependencies]` and `cargo build`:
```bash
$ otool -L target/debug/librust_input.dylib
target/debug/librust_input.dylib:
        /XXX/zxy/target/debug/deps/librust_input.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
        /usr/lib/libresolv.9.dylib (compatibility version 1.0.0, current version 1.0.0)
```

Ah, for some reason we picked up a dependency on libstd shared library when we added the zxy crate.

Tried a couple of things:
- Added `#![no_std]` to the top of `zxy/Cargo.toml`
- Changed zxy crate to `crate-type = ["dylib"]`

But the dependency remains.  We ended up setting `LD_LIBRARY_PATH`, but I suspect there's a better (i.e. more "correct") way to deal with this.  I'm using VS Code, so in `launch.json`:

```json
"configurations": [
    {
        "env": {
            "LD_LIBRARY_PATH": "/XXX/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/"
        }
    }
]
```

## The Thrift Connection

Create `input.thrift`:
```thrift
namespace * zxy.SDK.Input

service Input {
    bool Test(),
}
```

Defining a Thrift `service` can be used to create [request-response](https://en.wikipedia.org/wiki/Request%E2%80%93response) [client-server](https://en.wikipedia.org/wiki/Client%E2%80%93server_model) RPC.

Generate .NET Core and Rust bindings:
```bash
thrift -gen netcore -out . thrift/input.thrift
thrift -gen rs -out src thrift/input.thrift
```

C# "server":
```csharp
public class InputXy : BackgroundService
{
    [DllImport("rust_input")]
    static extern int start();

    public InputXy(IConfiguration configuration, ILogger<InputXy> logger, IZxyContext context)
    {
        var processor = new zxy.SDK.Input.Input.AsyncProcessor(new Processor(logger));
        zxyContext.RegisterProcessor(InputPlugin.ServiceName, processor);
        //...
    }

    //...
}

class Processor : zxy.SDK.Input.Input.IAsync
{
    public Processor(ILogger logger)
    {
        this.logger = logger;
    }
    public Task<bool> TestAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Test");
        return Task.FromResult(true);
    }
    ILogger logger;
}
```

In rust_input crate, create our Thrift client ([similar to before]({% post_url /2018/2018-08-30-rust-thrift %})):
```rust
extern crate zxy;
extern crate thrift;

use thrift::protocol::{TBinaryInputProtocol, TBinaryOutputProtocol, TMultiplexedOutputProtocol};
use thrift::transport::{TTcpChannel, TIoChannel};
use zxy::input::TInputSyncClient;

fn do_thrift(settings: &AppSettings) {
    // Create TCP transport "channel" to local server
    let mut channel = TTcpChannel::new();
    println!("Connecting to TCP...");
    channel.open(&format!("127.0.0.1:{}", settings.zxy.api_bridge.TCPport)).unwrap();

    // Decompose TCP channel into read/write-halves for in/out protocols
    let (readable, writable) = channel.split().unwrap();
    let in_proto = TBinaryInputProtocol::new(readable, true);
    let out_proto = TBinaryOutputProtocol::new(writable, true);

    // Multiple clients can be multiplexed over a single transport
    let out_proto = TMultiplexedOutputProtocol::new("input", out_proto);

    // Initialize the client
    let mut client = zxy::input::InputSyncClient::new(in_proto, out_proto);

    // RPC to the "server"
    println!("RPC...");
    client.test().unwrap();
    println!("DONE!");
}
```

And... the C# server emits the sweet, sweet log output we expect:
```
info: zxy.zxy0.InputXy[0]
      Test
```

Just to be clear, this doesn't use NNG; it's Thrift over TCP.

## To Be Continued...

One thing that's fantastic about our microservice architecture is once we start a Rust service we can interact with it the same as our C# services (i.e. via Thrift RPC or NNG pub/sub).  No need to deal with managed to unmanaged interop which [gets pretty hairy for non-trivial types]({% post_url /2018/2018-08-21-windows-services %}#failure-actions).

First thing that came to mind was actually replacing the C# broker with a Rust implementation so [it isn't affected by that pesky garbage collector](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals#what-happens-during-a-garbage-collection).