---
layout: post
title: Rust & Apache Thrift
tags:
- rust
- thrift
---

Using [Apache Thrift](https://thrift.apache.org/) enables us to generate client libraries [for our SDK](https://github.com/subor/sdk) (still very-WIP) targetting [a variety of languages](https://thrift.apache.org/docs/Languages).  I'm going to create a test library for [Rust](https://www.rust-lang.org/en-US/) that makes a simple RPC call to our [background service]({% post_url /2018/2018-08-21-windows-services %}).

## Generation

One of the reasons we introduced [API Tool](https://github.com/subor/sdk/blob/master/docs/topics/build_sdk_source.md#thrift) was to make it easier to work with our [.thrift interface definition files](https://github.com/subor/sdk/tree/master/ThriftFiles):  
![]({{ "/assets/devtool_apitool_rs.png" | absolute_url }})

Click __Generate__ to process all the thrift files:
```
18/08/22 12:54:53  Info ApiTool --ThriftFiles="D:\ruyi\sdk\ThriftFiles" --ThriftExe="D:\ruyi\..\tools\thrift\thrift.exe" --CommonOutput="D:\ruyi\sdk\SDK.Gen.CommonAsync" --ServiceOutput="D:\ruyi\sdk\SDK.Gen.ServiceAsync" --Gen="rs" --Generate
18/08/22 12:54:53  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.ServiceAsync D:\ruyi\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKDataTypes.thrift
18/08/22 12:54:53  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.ServiceAsync D:\ruyi\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKServices.thrift
18/08/22 12:54:54  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.CommonAsync D:\ruyi\sdk\ThriftFiles\CommonType\CommonTypeSDKDataTypes.thrift
```

Note the `-gen rs ...` output showing calls to `thrift.exe`.

The particulars of our platform aren't important for this excercise.  You could substitute the [thrift tutorial](https://thrift.apache.org/tutorial/).

## Rust-y Baby Steps

As a first step I want to build a rust library containing the generated source files.

1. Start a new "subor" library:
    ```
    cargo new --lib subor
    ```
1. Launch [Visual Studio Code](https://code.visualstudio.com/) and install [Rust support](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust).  Open the `subor/` folder cargo created.

1. Copy all the generated .rs files into the `src/` directory.

Right next to `lib.rs`, `localization_service_s_d_k_data_types.rs` caught my eye and seems like a good place to start.  It contains:
```rust
impl LanguageChangedMsg {
  pub fn new<F1, F2>(new_language: F1, old_language: F2) -> LanguageChangedMsg where F1: Into<Option<String>>, F2: Into<Option<String>> {
    LanguageChangedMsg {
      new_language: new_language.into(),
      old_language: old_language.into(),
    }
  }
  //...
```

Which was generated from [LocalizationServiceSDKDataTypes.thrift](https://github.com/subor/sdk/blob/master/ThriftFiles/LocalizationService/LocalizationServiceSDKDataTypes.thrift):
```
struct LanguageChangedMsg {
    1: string newLanguage,
    2: string oldLanguage,
}
```

To bring that file into scope, to the top of `lib.rs` add:
```rust
mod localization_service_s_d_k_data_types;
```

Bring up the VS Code terminal with ```^` ``` (that's Ctrl+Backtick- or "grave accent" if you're fancy).

Build tests with `cargo build --tests`:
```
error[E0463]: can't find crate for `ordered_float`
 --> src/localization_service_s_d_k_data_types.rs:9:1
  |
9 | extern crate ordered_float;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^ can't find crate
```

Check [crates.io](https://crates.io/) for external dependencies.  To fix this and the next few errors, to `Cargo.ml` add:
```
[dependencies]
ordered-float = "0.5.0"
thrift = "0.0.4"
try_from = "0.2.2"
```

Build:
```
error[E0432]: unresolved import `ordered_float`
  --> src/localization_service_s_d_k_data_types.rs:13:5
   |
13 | use ordered_float::OrderedFloat;
   |     ^^^^^^^^^^^^^ Did you mean `self::ordered_float`?
```

The crates are `extern`'d in the sub-module (i.e. `localization_service_s_d_k_data_types.rs`), so you either do what it says and prepend `self::` everywhere (_ugh_).  Or, to the top of lib.rs add:
```rust
extern crate ordered_float;
extern crate thrift;
extern crate try_from;
```

Now, try using the `LanguageChangedMsg` type in the test:
```rust
#[cfg(test)]
mod tests {
    // Bring entire contents of module into scope
    use super::localization_service_s_d_k_data_types::*;
    #[test]
    fn it_works() {
      let msg = LanguageChangedMsg::new("stuff".to_owned(), "this".to_owned());
```
Use [glob operator](https://doc.rust-lang.org/book/2018-edition/ch07-03-importing-names-with-use.html) (also see [example with tests](https://doc.rust-lang.org/book/2018-edition/ch11-01-writing-tests.html#checking-results-with-the-assert-macro)) to bring everything in that file into scope.

Inside `it_works()` function type `let msg = L` (Note: capital __"L"__) and "intellisense" should suggest the rest.

Finally, `cargo test` to run the test and it should pass.

## Client

Confident we can build things, let's make a full-fledged client so we can do RPC.

The specification in `LocalizationServiceSDKServices.thrift`:
```
service LocalizationService {
  // ...
  string GetCurrentLanguage(),
  // ...
}
```

Generates `localization_service_s_d_k_services.rs`:
```rust
pub trait TLocalizationServiceSyncClient {
  // ...

  fn get_current_language(&mut self) -> thrift::Result<String>;

  // ...
}

impl <IP, OP> LocalizationServiceSyncClient<IP, OP> where IP: TInputProtocol, OP: TOutputProtocol {
  pub fn new(input_protocol: IP, output_protocol: OP) -> LocalizationServiceSyncClient<IP, OP> {
    LocalizationServiceSyncClient { _i_prot: input_protocol, _o_prot: output_protocol, _sequence_number: 0 }
  }
}

impl <C: TThriftClient + TLocalizationServiceSyncClientMarker> TLocalizationServiceSyncClient for C {
  // ...
  fn get_current_language(&mut self) -> thrift::Result<String> {
    // ...
```

This will make sense if you're familiar with thrift and rust:
- `TLocalizationServiceSyncClient` defines an interface to access a "service"
- It specifies several RPC calls including `get_current_language()`
- A client instance can be created with `LocalizationServiceSyncClient::new()` given an input and output protocol

[thrift::protocol module docs](https://docs.rs/thrift/0.0.4/thrift/protocol/index.html) show how to get started:
```rust
use thrift::protocol::{TBinaryInputProtocol, TBinaryOutputProtocol, TMultiplexedOutputProtocol};
use thrift::transport::{TTcpChannel, TIoChannel};

use super::localization_service_s_d_k_services::*;

#[test]
fn client() {
    // Create TCP transport "channel" to local server
    let mut channel = TTcpChannel::new();
    channel.open("127.0.0.1:11290").unwrap();

    // Decompose TCP channel into read/write-halves for in/out protocols
    let (readable, writable) = channel.split().unwrap();

    // These take ownership of their first argument, so using TCP channel 
    // directly would require multiple TCP connections
    let in_proto = TBinaryInputProtocol::new(readable, true);
    let out_proto = TBinaryOutputProtocol::new(writable, true);

    // Multiple clients can be multiplexed over a single transport.
    // The server side of our application is expecting "SER_xxx" to 
    // route to the correct service.
    let out_proto = TMultiplexedOutputProtocol::new("SER_L10NSERVICE", out_proto);

    let mut client = LocalizationServiceSyncClient::new(in_proto, out_proto);

    // RPC to server
    client.get_current_language().unwrap();
}
```

Bi-directional channels like `TTcpChannel` implement [TIoChannel::split()](https://docs.rs/thrift/0.0.4/thrift/transport/trait.TIoChannel.html) to create "readable" and "writable" halves.  Each binary protocol can then take ownership of its own half.

Wrap output protocol with [`TMultiplexedOutputProtocol`](https://docs.rs/thrift/0.0.4/thrift/protocol/struct.TMultiplexedOutputProtocol.html) so we can have multiple `T*SyncClient`s that share a single TCP connection (or other transport).  The first argument, `service_name`, is application-defined name given to the service- here `"SER_L10NSERVICE"`.  Although not a thrift requirement, the server side of our application is expecting it.

To do RPC, make request using client method.  If you check the generated C# and rust source code, notice:
- Rust methods use snake-case: `get_current_language()`
- `async` C# methods append `Async` suffix: `GetCurrentLanguageAsync()`
- Serialized messages specify method by name from the thrift specification: `GetCurrentLanguage`
- Multiplexing adds service name: `SER_L10NSERVICE:GetCurrentLanguage`


## Server

For the server-side I'm testing with [our latest release](https://github.com/subor/sdk/releases) of [layer0]({% post_url /2018/2018-08-21-windows-services %}).

Here's a compatible server in C# using the same thrift files:
```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var server = new Thrift.Transports.Server.TServerSocketTransport(11290);
        server.Listen();
        // Create service processor and register with multiplexor
        var mux = new TMultiplexedProcessor();
        var processor = new LocalizationService.AsyncProcessor(new SettingHandler());
        mux.RegisterProcessor("SER_L10NSERVICE", processor);
        while (true){
            // Accept client connection, wrap with protocol, and hand to multiplexor
            var client = await server.AcceptAsync();
            await Task.Run(async () =>
            {
                var protocol = new Thrift.Protocols.TBinaryProtocol(client);
                Console.WriteLine(await mux.ProcessAsync(protocol, protocol));
            });
        }
    }
}

class SettingHandler : LocalizationService.IAsync
{
    //...

    public Task<string> GetCurrentLanguageAsync(CancellationToken cancellationToken)
    {
        return Task.FromResult("en-US");
    }

    //...
}
```

When a client connects and sends a request `SER_L10NSERVICE:GetCurrentLanguage`:
1. `TMultiplexedProcessor.ProcessAsync()` extracts service name (`SER_L10NSERVICE`) and matches to `ITAsyncProcessor` added with `RegisterProcessor()`
1. `ProcessAsync()` of matched processor is called.  It extracts method name (`GetCurrentLanguage`) and matches to corresponding method.
1. Arguments (if any) are deserialized and method of `IAsync` handler instance passed to processor is called (`SettingHandler.GetCurrentLanguageAsync()`)

## Speed Bumps

Unfortunately it seems like Rust "intellisense" in VS Code isn't quite there yet.  After typing `client.` I expected `get_current_language()` et al. to be suggested, but only the `TThriftClient` plumbing appeared:  
![]({{ "/assets/vscode_rust_intellisense_fail.png" | absolute_url }}){:width="50%"}

If client RPC calls fail with:
```
ApplicationError { kind: WrongMethodName, message: "expected GetCurrentLanguage got SER_L10NSERVICE:GetCurrentLanguage" }', libcore/result.rs:945:5
```

Check the names registered with `TMultiplexedOutputProtocol::new()` (client) and `RegisterProcessor()` (server) match.

Overall, my experience with Rust was like my experience with F#; [if it compiles, it works](http://blog.tamizhvendan.in/blog/2013/08/08/if-your-fsharp-code-compiles-it-usually-works/).