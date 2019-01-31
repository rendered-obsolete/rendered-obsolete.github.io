---
layout: post
title: Rust `no_std`
tags:
- rust
- no_std
---

An earlier version of The Book had sections on no_std
- https://doc.rust-lang.org/1.6.0/book/using-rust-without-the-standard-library.html
- https://doc.rust-lang.org/1.6.0/book/no-stdlib.html

## Starting with Bindgen

Details are in [an earlier post]({% post_url /2018/2018-09-30-rust-ffi-ci %})

`build.rs`:
```rust
// https://rust-lang-nursery.github.io/rust-bindgen
// https://docs.rs/bindgen
let bindings = bindgen::Builder::default()
    .header("wrapper.h")
    // This is needed if use `#include <nng.h>` instead of `#include "path/nng.h"`
    //.clang_arg("-Inng/src/")
    .generate()
    .expect("Unable to generate bindings");
let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
bindings
    .write_to_file(out_path.join("bindings.rs"))
    .expect("Couldn't write bindings");
```

`lib.rs`:
```rust
// Suppress the flurry of warnings caused by using "C" naming conventions
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]
// Disable clippy since this is all bindgen generated code
#![allow(clippy::all)]

// This matches bindgen::Builder output
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

## no_std

Add `#![no_std]` to the top of `lib.rs` and build:
```rust
error[E0433]: failed to resolve: could not find `std` in `{{root}}`
    --> /Users/jake/projects/runng/target/debug/build/runng-sys-c34663584b4c622a/out/bindings.rs:2697:22
     |
2697 |     pub fp_offset: ::std::os::raw::c_uint,
     |                      ^^^ could not find `std` in `{{root}}`
```

Embedded Rust Book [mentions](https://rust-embedded.github.io/book/interoperability/c-with-rust.html#automatically-generating-the-interface) cty crate.

`lib.rs` add [`ctypes_prefix`](https://docs.rs/bindgen/0.47.0/bindgen/struct.Builder.html#method.ctypes_prefix) before calling `generate`:
```rust
let bindings = bindgen::Builder::default()
    .header("wrapper.h")
    // `::std::os::raw::` becomes `cty::`
    .ctypes_prefix("cty")
    .generate()
```

In `Cargo.toml`:
```toml
[dependencies]
cty = "0.1"
```

Build:
```rust
error[E0433]: failed to resolve: could not find `std` in `{{root}}`
    --> /Users/jake/projects/runng/target/debug/build/runng-sys-c34663584b4c622a/out/bindings.rs:1965:31
     |
1965 | pub type nng_aio_cancelfn = ::std::option::Option<
     |                               ^^^ could not find `std` in `{{root}}`
```

[`use_core`](https://docs.rs/bindgen/0.47.0/bindgen/struct.Builder.html#method.use_core):
```rust
let bindings = bindgen::Builder::default()
    .header("wrapper.h")
    .ctypes_prefix("cty")
    .use_core()
    .generate()
```

Build:
```rust
error[E0308]: mismatched types
  --> runng/src/socket.rs:42:17
   |
42 |                 argument,
   |                 ^^^^^^^^ expected enum `cty::c_void`, found enum `std::ffi::c_void`
   |
   = note: expected type `*mut cty::c_void`
              found type `*mut std::ffi::c_void`
```

[cty crate defines](https://github.com/japaric/cty/blob/4d8b2525ae6adcd5a2cd09ec575234f0f65a1a80/src/lib.rs#L123) `c_void`:
```rust
#[repr(u8)]
pub enum c_void {
    // Two dummy variants so the #[repr] attribute can be used.
    #[doc(hidden)]
    __variant1,
    #[doc(hidden)]
    __variant2,
}
```

We need cty to re-export `std::ffi::c_void`:
```rust
pub type c_void = core::ffi::c_void;
```


https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#testing-a-bugfix

This project is using [workspace](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) like the message says, `[path.crates.io]` needs to go in the top-level Cargo.toml.

```toml
# In runng-sys/Cargo.toml
[dependencies]
cty = "0.1"

# In Cargo.toml
[patch.crates-io]
cty = { git = 'https://github.com/jeikabu/cty' }
```

If you put `[path.crates.io]` in the sub-crate:
```
warning: patch for the non root package will be ignored, specify patch at the workspace root:
package:   /XXX/runng/runng-sys/Cargo.toml
workspace: /XXX/runng/Cargo.toml
```

Everything builds!

## So what?

`cargo build --release` and take a look at the binaries in `target/release/`:
- Almost the same size 891584 (no_std) vs. 892216 (std).
- Similar symbol tables (mac: `objdump -t librunng_sys.rlib`, linux: `readelf`)
- Similar shared library dependencies (mac: `objdump -macho -dylibs-used librunng_sys.rlib` or `otool -L librunng_sys.rlib`, linux: `ldd`)

I was more than a little disappointed it didn't have the same drastic impact as `crate-type = ["cdylib"]`.

