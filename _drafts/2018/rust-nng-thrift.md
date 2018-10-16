---
layout: post
title: Rust Next
tags:
- rust
- nng
- thrift
- csharp
---

[Last time]({% post_url /2018/2018-09-30-rust-ffi-ci %}) I loaded a Rust binary as part of a NetCore application.  Now I'm going to get C# and Rust communicating with Nng and exchange a Thrift message.

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
    http: HttpSettings
}

#[derive(Deserialize,Debug)]
struct HttpSettings {
    port: u16
}

fn main() {
    let file = File::open("/absolute_project_path/appsettings.json").unwrap();
    // Deserialize AppSettings from file
    let settings: AppSettings = serde_json::from_reader(file).unwrap();
    println!("{:?}", settings);
}
```

Importance of location of `#[macro_use]` comes from [this SO](https://stackoverflow.com/questions/29068716/how-do-you-use-a-macro-from-inside-its-own-crate).

## Thrift

Which is ok, but it's going to get tedious (and error-prone) to maintain structs here (as well as in C#).
