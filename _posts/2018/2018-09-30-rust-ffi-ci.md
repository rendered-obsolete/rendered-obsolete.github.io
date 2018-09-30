---
layout: post
title: Rust FFI crate and CI
tags:
- rust
- ffi
- devops
- appveyor
- travis
- crates.io
---

For the next phase of a project I need Rust bindings to [nng](https://github.com/nanomsg/nng).  In this post I'll create a Rust crate with the bindings and setup Appveyor to do Windows builds and Travis to do OSX/Linux builds.

Source code: [Github/jeikabu/runng-sys](https://github.com/jeikabu/runng-sys).

## Rust FFI

Foreign Function Interface (FFI) is the way you can call [to/from Rust and another programming language](https://github.com/alexcrichton/rust-ffi-examples).  It is akin to Java's [Java-Native-Interface (JNI)](https://en.wikipedia.org/wiki/Java_Native_Interface), .Net's [C++/CLI](https://en.wikipedia.org/wiki/C%2B%2B/CLI) or [PInvoke](https://docs.microsoft.com/en-us/cpp/dotnet/how-to-call-native-dlls-from-managed-code-using-pinvoke), and other mechanisms in many other programming languages.

[Bindgen](https://github.com/rust-lang-nursery/rust-bindgen) can automatically generate Rust FFI bindings to C (and some C++) libraries.  Luckily, nng is C.  [This blog post](http://fitzgeraldnick.com/2016/12/14/using-libbindgen-in-build-rs.html) is referenced several places and looks like the origin of the [bindgen tutorial](https://rust-lang-nursery.github.io/rust-bindgen).

But first we need a library for rustc to link to.  We get lucky again because nng uses [cmake](https://cmake.org/) and 
[cmake-rs](https://github.com/alexcrichton/cmake-rs) allows us to run it from Rust.

`cargo new --lib runng-sys` to create a new project and in `./Cargo.toml`:
```toml
[build-dependencies]
cmake = "0.1"
bindgen = "0.40"
```

Following the bindgen tutorial, `./wrapper.h` includes nng header files:
```c
#include "nng/src/nng.h"

// protocols
#include "nng/src/protocol/bus0/bus.h"
#include "nng/src/protocol/pipeline0/pull.h"
#include "nng/src/protocol/pipeline0/push.h"
#include "nng/src/protocol/pubsub0/pub.h"
#include "nng/src/protocol/pubsub0/sub.h"
#include "nng/src/protocol/reqrep0/rep.h"
#include "nng/src/protocol/reqrep0/req.h"

// transports
#include "nng/src/transport/inproc/inproc.h"
#include "nng/src/transport/ipc/ipc.h"
#include "nng/src/transport/tcp/tcp.h"
#include "nng/src/transport/ws/websocket.h"
```

`./build.rs` will be built and executed before the rest of the crate:
```rust
extern crate bindgen;
extern crate cmake;

use cmake::Config;
use std::{
    env,
    path::PathBuf,
};

fn main() {
    // Run cmake to build nng
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
    // Tell rustc to use nng static library
    println!("cargo:rustc-link-lib=static=nng");

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

`src/lib.rs` contains:
```rust
// Suppress the flurry of warnings caused by using "C" naming conventions
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

// This matches bindgen::Builder output
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

Run `cargo build` to:

1. Use cmake and [ninja](https://ninja-build.org/) to build nng.  This is similar to running:
    
        cd nng && cmake -G Ninja -DCMAKE_BUILD_TYPE=Release . && ninja install

1. Run bindgen to generate native library bindings in `$OUT_DIR/bindings.rs`

Notes:

- [cmake docs](https://docs.rs/cmake/) and [bindgen docs](https://docs.rs/bindgen).
- Ninja is the [recommended cmake generator](https://github.com/nanomsg/nng) for nng.
- Both Cmake and Ninja must be in `PATH` environment variable (or set `CMAKE` and `CMAKE_MAKE_PROGRAM`)
- Debug with `cargo build --verbose`
- Cmake generates static library: `target/CONFIG/build/runng-sys-abc123/out/lib/libnng.a` (on OSX and probably Linux).
- If decide to use `#include <nng.h>`, etc. instead, need to supply include path via `clang_arg()` (see this [SO](https://stackoverflow.com/questions/42741815/setting-the-include-path-with-bindgen)).
- `$OUT_DIR` is set by cargo.


## Test Binding

To write a simple test, need to get a look at what was just generated in VS Code (on OSX must first [Install 'code' command in PATH](https://code.visualstudio.com/docs/setup/mac#_launching-from-the-command-line)):
```bash
code `find . -name bindings.rs`
```

This will be "minified" Rust source code.  To make it readable, __View > Command Palette > `format document`__.

Write a simple nng request/reply client/server:
```rust
use std::ffi::CString;

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn it_works() {
        unsafe {
            let url = CString::new("inproc://test").unwrap();
            let url = url.as_bytes_with_nul().as_ptr() as *const i8;

            // Reply socket
            let mut rep_socket = nng_socket { id: 0 };
            assert_eq!(0, nng_rep0_open(&mut rep_socket));
            assert_eq!(0, nng_listen(rep_socket, url, std::ptr::null_mut(), 0));

            // Request socket
            let mut req_socket = nng_socket { id: 0 };
            assert_eq!(0, nng_req0_open(&mut req_socket));
            assert_eq!(0, nng_dial(req_socket, url, std::ptr::null_mut(), 0));

            // Send message
            let mut req_msg = nng_msg { _unused: [] };
            let mut req_msg = &mut req_msg as *mut nng_msg;
            assert_eq!(0, nng_msg_alloc(&mut req_msg, 0));
            // Add a value to the body of the message
            let val = 0x12345678;
            assert_eq!(0, nng_msg_append_u32(req_msg, val));
            assert_eq!(0, nng_sendmsg(req_socket, req_msg, 0));
            
            // Receive message with reply socket
            let mut recv_msg = nng_msg { _unused: [] };
            let mut recv_msg = &mut recv_msg as *mut nng_msg;
            assert_eq!(0, nng_recvmsg(rep_socket, &mut recv_msg, 0));
            // Remove our value from the body of the received message
            let mut recv_val: u32 = 0;
            assert_eq!(0, nng_msg_trim_u32(recv_msg, &mut recv_val));
            assert_eq!(val, recv_val);
            // Can't do this because nng uses network order (big-endian)
            //assert_eq!(val, *(nng_msg_body(recv_msg) as *const u32));

            nng_close(req_socket);
            nng_close(rep_socket);
        }
        
    }
}
```

Using [`CString`](https://doc.rust-lang.org/std/ffi/struct.CString.html) to create null-terminated strings as suggested by [this SO](https://stackoverflow.com/questions/28649311/what-is-the-proper-way-to-go-from-a-string-to-a-const-i8).  To get a `const char*` pointer to `"abc\0"` in Rust requires:
```rust
let cstring = std::ffi::CString::new("abc").unwrap().as_bytes_with_nul().as_ptr() as *const i8;
```

Struggled with uses of `Type**` in the C.  For example, [`nng_msg_alloc()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg_alloc.3.html) binding is:
```rust
pub fn nng_msg_alloc(arg1: *mut *mut nng_msg, arg2: usize) -> ::std::os::raw::c_int;
```

To get `*mut *mut`:
```rust
// NO; can't convert &mut &mut to *mut *mut
let msg = &mut &mut recv_msg as *mut *mut nng_msg;
// Yes
let msg = &mut (&mut recv_msg as *mut nng_msg) as *mut *mut nng_msg;
// Yes
let msg = &mut (&mut recv_msg as *mut nng_msg);
```

The last line is a relief, `as *mut *mut type` is a word-y cast.

## Appveyor

Already using [Appveyor](https://www.appveyor.com/) on [another project]({% post_url /2018/2018-09-21-dotnet %}#appveyor), so decided to start with that.

Using [this `appveyor.yml` for Rust](https://github.com/starkat99/appveyor-rust) as a starting point:
```yml
image: 
- Visual Studio 2017
- Ubuntu

# Build configurations
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

# Platform-specific configuration
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
  # Run only our tests (bindgen also generates tests)
  - cargo test --verbose -- "tests::"
```

All the `matrix:` stuff is unfortunate.  I tried to place the `environment` node inside `for`, but it didn't result in build permutations.  So, moved it out and `matrix: exclude:` to remove Windows builds on Linux, and vice versa. 

Use `for` node to have [different configuration for each build matrix job](https://www.appveyor.com/docs/build-configuration/#specializing-matrix-job-configuration).


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

This is caused by [a recent change](https://github.com/alexcrichton/cmake-rs/pull/57) that adds `-Thost=x64` when compiling with `x86_64-pc-windows-msvc`.  [Submitted a PR](https://github.com/alexcrichton/cmake-rs/pull/63) to avoid `-Thost` with Ninja.

The `i686-pc-windows-msvc` (32-bit) build gets further failing with:
```
CMake Warning:
  Manually-specified variables were not used by the project:
    CMAKE_CXX_COMPILER
thread 'main' panicked at 'Unable to find libclang: "couldn\'t find any of [\'clang.dll\', \'libclang.dll\'], set the LIBCLANG_PATH environment variable to a path where one of these files can be found (skipped: [(C:\\Program Files\\LLVM\\bin\\libclang.dll: invalid DLL (64-bit))])"', libcore\result.rs:945:5
```

[llvm is already installed](https://www.appveyor.com/docs/windows-images-software/#llvm) and it found `C:\Program Files\LLVM\bin\libclang.dll`, but it's 64-bit when we need 32-bit.  Looks like I'll have to manually install 32-bit llvm...

## Travis

In the course of investigating Rust code-coverage I found out [Travis CI](https://travis-ci.org/) has [explicit support for Rust](https://docs.travis-ci.com/user/languages/rust/).

Can easily configure a build without messing with rustup-init by using a `.travis.yml` similar to:
```yml
language: rust
rust:
  - stable
  #- nightly

os:
  - linux
  - osx

sudo: false
addons:
  apt:
    packages:
      # To build nng
      - cmake
      - ninja-build

script:
  - cargo build
  - cargo test -- "tests::"
```

I explicitly avoid `sudo` to enable a [containerized environment](https://docs.travis-ci.com/user/reference/overview/#Virtualization-environments).  In particular, using [`addons: apt:`](https://docs.travis-ci.com/user/installing-dependencies/#installing-packages-on-container-based-infrastructure) to install packages instead of `sudo apt-get install`.

Use `-- "tests::"` with `cargo test` to filter out bindgen's tests.  One of those tests fails on Travis for some reason.

This build completes quickly; takes just over 2 minutes while Appveyor takes more than 4 (mostly to install clang/llvm).

## Code Coverage

Things get messy with code coverage.

There's two good posts detailing Rust CI with Travis: [first](https://sunjay.ca/2016/07/25/rust-code-coverage), [second](https://medium.com/@Razican/continuous-integration-and-code-coverage-report-for-a-rust-project-5dfd4d68fbe5).  Both are worth reading and detail using [kcov](https://github.com/SimonKagstrom/kcov) for code-coverage.  Travis uses Ubuntu 14.04 (Trusty Tahr) so the version of kcov installable with apt is too old to work with Rust.

In `.travis.yml`:
```yml
language: rust
rust:
  - stable

os:
  - linux
  - osx

# Force full VM (kcov doesn't work in default container environment)
sudo: required
addons:
  apt:
    packages:
      # To build nng
      - cmake
      - ninja-build
      # To build kcov
      - libcurl4-openssl-dev
      - libelf-dev
      - libdw-dev
      - gcc
      - binutils-dev
      - libiberty-dev

before_install:
  # Using `source` so can update PATH environment variable
  - source ./scripts/install.sh

script:
  - cargo build
  - cargo test -- "tests::"

after_success:
  - ./scripts/after_success.sh
```

`install.sh` manually installs ninja on OSX to [avoid using homebrew](https://docs.travis-ci.com/user/reference/osx#homebrew):
```bash
#!/usr/bin/env bash

if [[ "$TRAVIS_OS_NAME" == "osx" ]]; then
    # `brew install ninja` requires `brew update` which takes ages....
    wget https://github.com/ninja-build/ninja/releases/download/v1.8.2/ninja-mac.zip
    unzip ninja-mac.zip
    export PATH=`pwd`:$PATH
fi
```

`after_success` step runs code-coverage [only on successful builds](https://docs.travis-ci.com/user/customizing-the-build/#the-build-lifecycle).  `after_success.sh` comes from [codecov example for Rust and travis](https://github.com/codecov/example-rust) (also see [kcov notes on codecov](https://github.com/SimonKagstrom/kcov/blob/master/doc/codecov.md)):
```bash
#!/usr/bin/env bash

TARBALL="v36.tar.gz"
KCOV_DIR="kcov-36"

if [[ "$TRAVIS_OS_NAME" == "linux" ]]; then
    # https://github.com/codecov/example-rust
    wget https://github.com/SimonKagstrom/kcov/archive/$TARBALL
    tar xzf $TARBALL
    cd $KCOV_DIR
    mkdir build
    cd build
    cmake ..
    make
    make install DESTDIR=../../kcov-build
    cd ../../
    rm -rf $KCOV_DIR
    for file in target/debug/runng_sys-*[^\.d]; do 
        mkdir -p "target/cov/$(basename $file)"
        # Arguments at the end are what would be passed to `cargo test`
        ./kcov-build/usr/local/bin/kcov --exclude-pattern=/.cargo,/usr/lib --verify "target/cov/$(basename $file)" "$file" -- "tests::"
    done
    # Upload reports in current directory
    # https://github.com/SimonKagstrom/kcov/blob/master/doc/codecov.md
    bash <(curl -s https://codecov.io/bash)
fi
```

This is a little different than the [codecov example](https://github.com/codecov/example-rust) :

1. Install a kcov release rather than github master
1. Avoid long chain using `&&`
1. Passing `-- "tests::"` args like when run `cargo test`

On the first run the coverage report was empty and the build log contained:
```
Can't set personality: Operation not permitted
kcov: error: Can't start/attach to /home/travis/build/jeikabu/runng/target/debug/runng_sys-ee3c607355e4b215
Child hasn't stopped: ff00
kcov: error: Can't start/attach to /home/travis/build/jeikabu/runng/target/debug/runng_sys-ee3c607355e4b215
```

See [this github issue](https://github.com/SimonKagstrom/kcov/issues/151) (and [the blog mentioned above](https://sunjay.ca/2016/07/25/rust-code-coverage#docker-security-settings)).  Basically, kcov doesn't work in containers unless docker is run with specific options.  There doesn't seem to be way to make Travis do that, so `sudo: required` uses a full VM and avoids the issue.

## Crates with Flair

Add [packaging and project meta-data](https://doc.rust-lang.org/cargo/reference/manifest.html) to `Cargo.toml`:
```toml
[package]
name = "runng-sys"
version = "0.1.1"
description = "Bindings to nng (Nanomsg-Next-Generation) aka Nanomsg2"
keywords = ["nng", "nanomsg"]
license = "MIT"
repository = "https://github.com/jeikabu/runng-sys"

[badges]
appveyor = { repository = "jake-ruyi/runng-sys", branch = "master", service = "github" }
travis-ci = { repository = "jeikabu/runng-sys", branch = "master" }
codecov = { repository = "jeikabu/runng-sys", branch = "master", service = "github" }
```

The `repository` values being the `OWNER/PROJECT` values from [Appveyor](https://www.appveyor.com/), [Travis](https://travis-ci.org/), and [codecov](https://codecov.io), respectively.

`cargo publish` packages the binding as a crate and uploads it to [crates.io](https://crates.io/crates/runng-sys):

![]({{ "/assets/crates-runng-sys.png" | absolute_url }})

## Fin

I'm tempted to run kcov on OSX so I can use the containerized Linux environment.  But I question the wisdom of running kcov on a "less recommended" platform simply for the neat-o factor.

Would like to put together [a PR](https://github.com/kt10/nng-rs) for [nng-sys crate](https://crates.io/crates/nng-sys).

Need to make a high-level wrapper around the nng bindings to hide the unsafe/pointer shenanigans.  Will likely model it after [nng.NETCore](https://github.com/subor/nng.NETCore).