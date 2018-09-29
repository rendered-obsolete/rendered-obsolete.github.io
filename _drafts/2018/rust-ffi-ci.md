---
layout: post
title: Rust ffi, crates, and CI
tags:
- rust
- devops
---

## nng ffi

[This blog post](http://fitzgeraldnick.com/2016/12/14/using-libbindgen-in-build-rs.html) is referenced several places and looks like the origin of the [bindgen tutorial](https://rust-lang-nursery.github.io/rust-bindgen).

https://docs.rs/bindgen

Full source at [Github/jeikabu/runng](https://github.com/jeikabu/runng).

In `./Cargo.toml`:
```
[build-dependencies]
cmake = "0.1"
bindgen = "0.40"
```


`./wrapper.h` includes nng header files:
```c
#include "nng/src/nng.h"
```

In `./build.rs`:
```rust
extern crate bindgen;
extern crate cmake;

use cmake::Config;
use std::{
    env,
    path::PathBuf,
};

fn main() {
    // https://docs.rs/cmake/
    let dst = Config::new("nng")
        .generator("Ninja")
        .define("CMAKE_BUILD_TYPE", "Release")
        .define("NNG_TESTS", "OFF")
        .define("NNG_TOOLS", "OFF")
        .build();
    
    // Check output of `cargo build --verbose`, should see something like:
    // -L native=/path/runng/target/debug/build/runng-sys-abc1234/out
    // That contains output from cmake
    println!("cargo:rustc-link-search=native={}", dst.join("lib").display());
    println!("cargo:rustc-link-lib=static=nng");

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
}
```

Run `cargo build` to:

1. Use cmake and [ninja](https://ninja-build.org/) to build nng.  This is similar to running:
    
        cd nng && cmake -G Ninja -DCMAKE_BUILD_TYPE=Release . && ninja install

1. Run bindgen to generate native library bindings to `$OUT_DIR/bindings.rs`

Notes:

- Debug with `cargo build --verbose`
- Ninja is the [recommended cmake generator](https://github.com/nanomsg/nng) for nng.
- Cmake output ends up in `target/CONFIG/build/runng-sys-abc123/out` and the static library is `lib/libnng.a`.
- If decide to use `#include <nng.h>` instead, need to supply include path via `clang_arg()` (see this [SO](https://stackoverflow.com/questions/42741815/setting-the-include-path-with-bindgen)).
- `$OUT_DIR` is set by cargo.

`src/lib.rs` contains:
```rust
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

// This matches bindgen::Builder output
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

The `#![allow()]` attributes suppress the flurry of warnings caused by using "c" naming conventions in Rust.

https://doc.rust-lang.org/cargo/reference/manifest.html

## Appveyor

Using [this `appveyor.yml`](https://github.com/starkat99/appveyor-rust) as a starting point:
```yml
image: 
- Visual Studio 2017
- Ubuntu

environment:
  matrix:
    # Stable 64-bit MSVC
    - target: x86_64-pc-windows-msvc
      channel: stable
    # Stable 32-bit MSVC
    - target: i686-pc-windows-msvc
      channel: stable
    # Dummy target so linux always has something to build
    - target: ubuntu-dummy
      channel: stable

matrix:
  exclude:
    # Linux ignore windows builds
    - image: Ubuntu
      target: x86_64-pc-windows-msvc
    - image: Ubuntu
      target: i686-pc-windows-msvc
    # Windows should ignore dummy linux configuration
    - image: Visual Studio 2017
      target: ubuntu-dummy

for:
-
  matrix:
    only:
      - image: Visual Studio 2017
  install:
    # Download rustup and install rust
    - appveyor DownloadFile https://win.rustup.rs/ -FileName rustup-init.exe
    - rustup-init -yv --default-toolchain %channel% --default-host %target%
    - set PATH=%PATH%;%USERPROFILE%\.cargo\bin
    # Install ninja (used to build nng)
    - choco install ninja
-
  matrix:
    only:
      - image: Ubuntu
  install:
    - sudo apt-get update
    # Need cmake/ninja/clang to build nng
    - sudo apt-get --yes install cmake ninja-build build-essential clang-3.9
    # Download and run rustup to install Rust (need "-y" to avoid waiting for input)
    - curl https://sh.rustup.rs -sSf > rustup-init.sh
    - sh rustup-init.sh -y
    # Add cargo to PATH
    - source $HOME/.cargo/env

# Skip build step since `cargo test` does it
build: false

test_script:
  # Run only our tests (not tests in dependencies)
  - cargo test --verbose -- "tests::"
```

The `environment: matrix:` is unfortunate.  I tried to place it inside `for: matrix:` but doesn't result in build permutations.  So, moved it out and have to exclude 

https://www.appveyor.com/docs/build-configuration/#specializing-matrix-job-configuration

The `x86_64-pc-windows-msvc` build fails with:
```
--- stderr
CMake Error at CMakeLists.txt:30 (project):
  Generator
    Ninja
  does not support toolset specification, but toolset
    host=x64
  was specified.
```

Sure enough, above that is:
```
running: "cmake" "C:\\projects\\runng\\nng" "-Thost=x64" "-G" "Ninja" ...
```

No idea where that `-Thost` argument comes from, doesn't seem to be any way to override it.

The `i686-pc-windows-msvc` build gets further failing with:
```
CMake Warning:
  Manually-specified variables were not used by the project:
    CMAKE_CXX_COMPILER
thread 'main' panicked at 'Unable to find libclang: "couldn\'t find any of [\'clang.dll\', \'libclang.dll\'], set the LIBCLANG_PATH environment variable to a path where one of these files can be found (skipped: [(C:\\Program Files\\LLVM\\bin\\libclang.dll: invalid DLL (64-bit))])"', libcore\result.rs:945:5
```

## Travis

In the course of investigating Rust code-coverage I found out [Travis](https://travis-ci.org/) has [explicit support for Rust](https://docs.travis-ci.com/user/languages/rust/).


Container apt
https://docs.travis-ci.com/user/installing-dependencies/#installing-packages-on-container-based-infrastructure

Almost instantaneously and finish in a little over 2 minutes while appveyor takes more than 4 (mostly to install clang/llvm).

## Code Coverage

Things get messy with code coverage.

Travis uses Ubuntu 14.04 (Trusty Tahr) so the version of kcov installable with apt is too old to work with Rust.

Found a few posts:
https://sunjay.ca/2016/07/25/rust-code-coverage
https://medium.com/@Razican/continuous-integration-and-code-coverage-report-for-a-rust-project-5dfd4d68fbe5

They're worth reading, but you probably want to use the script suggestions from:

- [codecov example for Rust and travis](https://github.com/codecov/example-rust)
- [kcov notes on codecov](https://github.com/SimonKagstrom/kcov/blob/master/doc/codecov.md)

In particular avoiding `sudo` to ensure we're running in a [containerized environment](https://docs.travis-ci.com/user/reference/overview/#Virtualization-environments).

`after_success` so code-coverage is [only run on succcessful builds](https://docs.travis-ci.com/user/customizing-the-build/#the-build-lifecycle).

```
Can't set personality: Operation not permitted
kcov: error: Can't start/attach to /home/travis/build/jeikabu/runng/target/debug/runng_sys-ee3c607355e4b215
Child hasn't stopped: ff00
kcov: error: Can't start/attach to /home/travis/build/jeikabu/runng/target/debug/runng_sys-ee3c607355e4b215
```

[github issue](https://github.com/SimonKagstrom/kcov/issues/151) (and [the blog mentioned above](https://sunjay.ca/2016/07/25/rust-code-coverage#docker-security-settings)).  Basically, kcov doesn't work in containerized environment unless docker is run with specific options.  Doesn't seem to be way to do that so `sudo: required` uses a full VM.