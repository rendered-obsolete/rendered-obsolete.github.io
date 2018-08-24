---
layout: post
title: Rust & Apache Thrift
tags:
- rust
- thrift
- zplus
---

Using [Apache Thrift](https://thrift.apache.org/) enables us to generate client libraries [for our SDK](https://github.com/subor/sdk) (still very-WIP) targetting [a variety of languages](https://thrift.apache.org/docs/Languages).  I'm going to create a test library for [Rust](https://www.rust-lang.org/en-US/) that makes a simple RPC call to our [background service]({% post_url /2018/2018-08-21-windows-services %}).

## Generation

One of the reasons we introduced our [API Tool](https://github.com/subor/sdk/blob/master/docs/topics/build_sdk_source.md#thrift) was to make it easier to work with our .thrift interface definition files:  
![]({{ "/assets/devtool_apitool_rs.png" | absolute_url }})

Click __Generate__ to process all the thrift files:
```
18/08/22 12:54:53  Info ApiTool --ThriftFiles="D:\ruyi\sdk\ThriftFiles" --ThriftExe="D:\ruyi\..\tools\thrift\thrift.exe" --CommonOutput="D:\ruyi\sdk\SDK.Gen.CommonAsync" --ServiceOutput="D:\ruyi\sdk\SDK.Gen.ServiceAsync" --Gen="rs" --Generate
18/08/22 12:54:53  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.ServiceAsync D:\ruyi\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKDataTypes.thrift
18/08/22 12:54:53  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.ServiceAsync D:\ruyi\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKServices.thrift
18/08/22 12:54:54  Info -gen rs -out D:\ruyi\sdk\SDK.Gen.CommonAsync D:\ruyi\sdk\ThriftFiles\CommonType\CommonTypeSDKDataTypes.thrift
```

Note the `-gen rs ...` output showing how we call `thrift.exe`.

The particulars of our platform aren't important for this excercise.  You could substitute the [thrift tutorial](https://thrift.apache.org/tutorial/).

## Rust-y Baby Steps

1. Start a new "subor" library:
    ```
    cargo new --lib subor
    ```
1. Launch [Visual Studio Code](https://code.visualstudio.com/) and install [Rust support](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust).  Open the `subor/` folder cargo created.

1. Copy all the generated .rs files into the `src/` directory.

Right next to `lib.rs`, `localization_service_s_d_k_data_types.rs` catches my eye:
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

To bring that file into scope, to the top of `lib.rs` add:
```rust
mod localization_service_s_d_k_data_types;
```

Bring up the VS Code terminal with ```^` ``` (that's Ctrl+Backtick- or "grave accent" if you're fancy).

Build with `cargo build`:
```
error[E0463]: can't find crate for `ordered_float`
 --> src/localization_service_s_d_k_data_types.rs:9:1
  |
9 | extern crate ordered_float;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^ can't find crate
```

Check [crates.io](https://crates.io/).  To fix the next few errors, to Cargo.ml add:
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

The crates are `extern`ed in the sub-module, so you either do what it says and prepend `self::` everywhere (_ugh_).  Or, to the top of lib.rs add:
```rust
extern crate ordered_float;
extern crate thrift;
extern crate try_from;
```

Tried using the `LanguageChangedMsg` type in the test:
```rust
#[cfg(test)]
mod tests {
    // Bring entire contents of module into scope
    use super::localization_service_s_d_k_data_types::*;
    #[test]
    fn it_works() {
```
Use [glob operator](https://doc.rust-lang.org/book/2018-edition/ch07-03-importing-names-with-use.html) (also see [example with tests](https://doc.rust-lang.org/book/2018-edition/ch11-01-writing-tests.html#checking-results-with-the-assert-macro)) to bring everything in that file into scope.

Inside `it_works()` function type `let msg = L` (Note: capital __"L"__) and "intellisense" should work.  After adding some arguments:
```rust
let msg = LanguageChangedMsg::new("stuff".to_owned(), "this".to_owned());
```

Again, `cargo build` and `cargo test` and everything should be ok.

## Client

The specification in `LocalizationServiceSDKServices.thrift`:
```
service LocalizationService {
  // ...
  string GetString(1: string key, 2: string context, 3: string language),
  // ...
}
```

Generates `localization_service_s_d_k_services.rs`:
```rust
pub trait TLocalizationServiceSyncClient {
  // ...

  /// Get a localized string.
  /// Return: localized string.
  fn get_string(&mut self, key: String, context: String, language: String) -> thrift::Result<String>;

  // ...
}

impl <IP, OP> LocalizationServiceSyncClient<IP, OP> where IP: TInputProtocol, OP: TOutputProtocol {
  pub fn new(input_protocol: IP, output_protocol: OP) -> LocalizationServiceSyncClient<IP, OP> {
    LocalizationServiceSyncClient { _i_prot: input_protocol, _o_prot: output_protocol, _sequence_number: 0 }
  }
}

impl <C: TThriftClient + TLocalizationServiceSyncClientMarker> TLocalizationServiceSyncClient for C {
  // ...
  fn get_string(&mut self, key: String, context: String, language: String) -> thrift::Result<String> {
    // ...
  }
  // ...
}
```

Propbably makes sense if you're already familiar with thrift and rust:
- `TLocalizationServiceSyncClient` defines an interface to access a "service"
  - It specifies a `get_string()` RPC call
- An instance can be initialized with `LocalizationServiceSyncClient::new()` given an input and output protocol

The [thrift::protocol module docs](https://docs.rs/thrift/0.0.4/thrift/protocol/index.html) show how to get started:
```rust
use thrift::protocol::{TBinaryInputProtocol, TBinaryOutputProtocol, TMultiplexedOutputProtocol};
use thrift::transport::{TTcpChannel, TIoChannel};

#[test]
fn client() {
    // Create TCP transport "channel"
    let mut channel = TTcpChannel::new();
    channel.open("127.0.0.1:9090").unwrap();

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
    client.get_string("test".to_owned(), String::default(), String::default()).unwrap();
}
```

Bi-directional channels like `TTcpChannel` implement [TIoChannel::split()](https://docs.rs/thrift/0.0.4/thrift/transport/trait.TIoChannel.html) to create "readable" and "writable" halves.  Each binary protocol can then take ownership of its own half.

Wrap output protocol with [`TMultiplexedOutputProtocol`](https://docs.rs/thrift/0.0.4/thrift/protocol/struct.TMultiplexedOutputProtocol.html) so we can have multiple `T*SyncClient`s that share a single TCP connection (or other transport).  The first argument, `service_name`, is application-defined name given to the service- here `"SER_L10NSERVICE"`.  Although not a thrift requirement, the server side of our application is expecting it.

Unfortunately it seems like Rust "intellisense" in VS Code isn't quite there yet.  After typing `client.` I expected `get_string()` et al to be suggested, but only the `TThriftClient` plumbing appeared:  
![]({{ "/assets/vscode_rust_intellisense_fail.png" | absolute_url }}){:width="50%"}