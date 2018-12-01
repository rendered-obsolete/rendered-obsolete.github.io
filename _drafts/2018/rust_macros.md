---
layout: post
title: Rust Macros
tags:
- rust
- macros
---

## Origin Story

My first programming job had me doing a lot of stuff in C.  We used macros heavily along with conditional compilation, and the "magic" of [`#` and `##`](https://en.wikipedia.org/wiki/C_preprocessor#Special_macros_and_directives) enabled us to do all kinds of compile-time code generation.

My first full-time job was all C++.  Our technical lead had me read ["Modern C++ Design"](https://en.wikipedia.org/wiki/Modern_C++_Design) by Andrei Alexandrescu.  Template meta-programming blew my mind, it was as if I'd never really known C++ at all.

## Rust's Macro Story

## Nng

NNG has a straight-forward C API.  As opposed to the function overloading you would find in C++ (or many other languages), there's a collection of functions for teach type.  The `nng_socket` type has get and set option functions:
```c
int nng_getopt(nng_socket s, const char *opt, void *val, size_t *valszp);
int nng_getopt_bool(nng_socket s, const char *opt, bool *bvalp);
int nng_getopt_int(nng_socket s, const char *opt, int *ivalp);
// ...
int nng_setopt(nng_socket s, const char *opt, const void *val, size_t valsz);
int nng_setopt_bool(nng_socket s, const char *opt, int bval);
// ...
```

Similar functions exist for the [`nng_dialer` type](https://nanomsg.github.io/nng/man/v1.1.0/nng_dialer_getopt.3):
```c
int nng_dialer_getopt(nng_dialer d, const char *opt, void *val, size_t *valszp);
int nng_dialer_getopt_bool(nng_dialer d, const char *opt, bool *bvalp);
int nng_dialer_getopt_int(nng_dialer d, const char *opt, int *ivalp);
// ...
int nng_dialer_setopt(nng_dialer d, const char *opt, const void *val,
    size_t valsz);
int nng_dialer_setopt_bool(nng_dialer d, const char *opt, bool bval);
// ...
```

And likewise for [`nng_listener`](https://nanomsg.github.io/nng/man/v1.1.0/nng_listener_getopt.3) and [`nng_pipe`](https://nanomsg.github.io/nng/man/v1.1.0/nng_pipe_getopt.3).

Wrapping the bindgen generated functions in our [runng-sys crate](https://crates.io/crates/runng-sys) is easy:
```rust
impl GetOpts for NngSocket {
    fn getopt_bool(&self, option: NngOption) -> NngResult<bool> {
        unsafe {
            let mut value: bool = Default::default();
            NngFail::succeed(nng_getopt_bool(self.socket, option.as_cptr(), &mut value), value)
        }
    }
    fn getopt_int(&self, option: NngOption) -> NngResult<i32> {
        unsafe {
            let mut value: i32 = Default::default();
            NngFail::succeed(nng_getopt_int(self.socket, option.as_cptr(), &mut value), value)
        }
    }
    // ...
}
```

I started to replicate this for `nng_listener`, `nng_dialer`, etc.

## Resources

Crates:
- [syn](https://docs.rs/syn)

Resources:
- [Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html) (`macro_rules!`)
- [syn examples](https://github.com/dtolnay/syn/tree/master/examples)
- https://cbreeden.github.io/Macros11/
- [Rust Programming Language](https://doc.rust-lang.org/book/2018-edition/appendix-04-macros.html)
- [The Reference](https://doc.rust-lang.org/reference/procedural-macros.html)