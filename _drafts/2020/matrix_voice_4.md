---
layout: post
title: 
tags:
- iot
- embedded
- rust
series: Smart Home
---


## Nightly

https://github.com/ctron/esp-idf-alloc
```rust
error: `#[panic_handler]` function required, but not found
error: `#[alloc_error_handler]` function required, but not found
```

However, it's [not yet stable](https://github.com/rust-lang/rust/issues/51540).
https://github.com/rust-lang/rust/blob/master/config.toml.example#L352

## Debugging

[Moving to LLVM 9.0 got debug info working](https://github.com/espressif/llvm-project/issues/10).  Meaning, among other compiling Rust projects with `--release` is no longer required.

https://mabez.dev/blog/posts/esp32-rust-svd-pac/

Install [Cortex-Debug](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug)

`.vscode/launch.json` https://github.com/MabezDev/xtensa-rust-quickstart/blob/master/.vscode/launch.json

1669528

```sh
# Install form, svd, and svd2rust
cargo install form
cargo install svd2rust
pip3 install --upgrade --user svdtools
```

If pip outputs the warning:
```
The script svd is installed in '/XXX/Python/3.7/bin' which is not on PATH.
```
Amend `PATH` environment variable:
```sh
export PATH=/XXX/Python/3.7/bin:$PATH
```
```
make
```

https://docs.rust-embedded.org/book/collections/
https://www.hackster.io/matrix-labs/program-over-the-air-on-esp32-matrix-voice-w-arduino-ide-5e76bb

https://docs.espressif.com/projects/esp-idf/en/latest/hw-reference/get-started-wrover-kit.html
