--- stderr
CMake Error: CMake was unable to find a build program corresponding to "Ninja".  CMAKE_MAKE_PROGRAM is not set.  You probably need to select a different build tool.
thread 'main' panicked at '
command did not execute successfully, got: exit code: 1

https://github.com/rust-lang/docs.rs

`0.4.0`
```
error: failed to select a version for the requirement `cargo_crates-io_docs-rs_test = "^0.4.0"`
  candidate versions found which didn't match: 0.3.0, 0.1.0+3
```

```
failed to parse the version requirement `0.4.0-` for dependency `cargo_crates-io_docs-rs_test`
```
```
failed to parse the version requirement `0.4.0-*` for dependency `cargo_crates-io_docs-rs_test`
```

## Specialization

https://play.rust-lang.org/?version=nightly&mode=debug&edition=2015
https://github.com/rust-lang/rust/issues/31844
https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md
https://stackoverflow.com/questions/34471212/how-to-implement-specialized-versions-of-a-generic-function

`Send` for pointer types
https://github.com/rust-lang/rust/issues/21709


## Conditional dependencies

https://github.com/alexcrichton/rfcs/blob/cargo-cfg-dependencies/text/0000-cargo-cfg-dependencies.md

```toml
# To enable bindgen only when building for PC, I'd like to have:
[target.'cfg(target_arch = "x86_64")'.dependencies]
runng-sys = { version = "1.1.1-rc", path = "../runng_sys", package = "nng-sys", features = ["build-bindgen"] }
[target.'cfg(target_arch = "aarch64")'.dependencies]
runng-sys = { version = "1.1.1-rc", path = "../runng_sys", package = "nng-sys" }
#
# But that doesn't work, `features` are set regardless of target arch.  
# Instead, could ignore the `build-bindgen` feature but that complicates everything:
cfg!(any(target_arch = "arm", target_arch = "aarch64"))
```

