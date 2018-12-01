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

In the .NET ecosystem there's [T4](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates) (which I've only used once), but we've done a lot of reflection and static code generation, and

## Nng

NNG has a straight-forward C API.  As opposed to the function overloading you would find in C++ (or many other languages), there's a collection of functions for each type.  The `nng_socket` type has get and set option functions:
```c
int nng_getopt_bool(nng_socket s, const char *opt, bool *bvalp);
int nng_getopt_int(nng_socket s, const char *opt, int *ivalp);
// ...
int nng_setopt(nng_socket s, const char *opt, const void *val, size_t valsz);
int nng_setopt_bool(nng_socket s, const char *opt, int bval);
// ...
```

Similar functions exist for the [`nng_dialer` type](https://nanomsg.github.io/nng/man/v1.1.0/nng_dialer_getopt.3):
```c
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

Already using helpers to deal with the boiler-plate:  
- `NngFail::succeed()` turns the C return value into one for Rust (`int -> Result<T>`)
- `.as_cptr()` turns an `enum` (with a `&'static [u8]`) into a C string naming the option (`NngOption -> *const i8`)

I started to replicate this for `nng_listener`, `nng_dialer`, et al.  But let's be honest, copy-pasting code is one of the most dubious of [code smells](https://en.wikipedia.org/wiki/Code_smell).

## Rust's Macro Story

We'd like to write something like:
```rust
#[derive(NngGetOpts,NngSetOpts)]
#[prefix = "nng_"]
#[member = "socket"]
pub struct NngSocket {
    socket: nng_socket,
}
```

Which would substitute the `prefix` and `member` values into `$PREFIX$getopt_bool(self.$MEMBER$, ..` to form `nng_getopt_bool(self.socket, ..`.  Yep, basically source code string substitution.

The end result being similar to:
```rust
impl GetOpts for NngSocket {
    fn getopt_bool(&self, option: NngOption) -> NngResult<bool> {
        nng_getopt_bool(self.socket, ..)
    }
    // ...
}
impl SetOpts for NngSocket {
    fn setopt_bool(&mut self, option: NngOption, value: bool) -> NngReturn {
        nng_setopt_bool(self.socket, ..)
    }
    // ...
}
```

`cargo new --lib runng_derive` and in `Cargo.toml`:
```toml
[package]
name = "runng_derive"
# ...

[lib]
proc-macro = true

[dependencies]
syn = "0.15"
quote = "0.6"
```

```rust
#![recursion_limit="128"]

extern crate proc_macro;
extern crate syn;
#[macro_use]
extern crate quote;

use proc_macro::TokenStream;
use syn::{
    Lit,
    Meta,
    MetaNameValue,
};

// Create #[derive(..)] macro
#[proc_macro_derive(NngGetOpts, attributes(prefix, member))]
pub fn derive_nng_get_opts(tokens: TokenStream) -> TokenStream {
    derive_nng_opts(tokens, gen_get_impl)
}
```

`recursion_limit` was recommended by the compiler after failing with:  
```
error: recursion limit reached while expanding the macro `stringify
```

`#[proc_macro_derive]` defines our derive macro.  Within that, `attributes(..)` defines our ["derive mode helper attributes"](https://doc.rust-lang.org/reference/procedural-macros.html#derive-mode-helper-attributes) (i.e. `#[prefix]` and `#[member]`).

First extract our `#[prefix ..]` and `#[member ..]` attributes:
```rust
fn derive_nng_opts<F>(tokens: TokenStream, gen_impl: F) -> TokenStream
    where F: Fn(&syn::Ident, String, String) -> TokenStream
{
    // Parse TokenStream into AST
    let ast: syn::DeriveInput = syn::parse(tokens).unwrap();
    let mut prefix: Option<String> = None;
    let mut member: Option<String> = None;
    // Iterate over the struct's #[...] attributes
    for option in ast.attrs.into_iter() {
        let option = option.parse_meta().unwrap();
        match option {
            // This matches `#[ident = lit]` attributes.  Match guard makes it `#[prefix = lit]`
            Meta::NameValue(MetaNameValue{ref ident, ref lit, ..}) if ident == "prefix" => {
                if let Lit::Str(lit) = lit {
                    prefix = Some(lit.value());
                }
            },
            // ...
        }
    }
    gen_impl(&ast.ident, prefix.unwrap(), member.unwrap())
}
```

Generate the code:
```rust
fn gen_get_impl(name: &syn::Ident, prefix: String, member: String) -> TokenStream {
    // Create function identifier like `nng_getopt_bool`
    let getopt_bool = prefix.clone() + "getopt_bool";
    let getopt_bool = syn::Ident::new(&getopt_bool, syn::export::Span::call_site());
    // ...

    let member = syn::Ident::new(&member, syn::export::Span::call_site());
    
    let gen = quote! {
        impl GetOpts for #name {
            fn getopt_bool(&self, option: NngOption) -> NngResult<bool> {
                unsafe {
                    let mut value: bool = Default::default();
                    NngFail::succeed( #getopt_bool (self.#member, option.as_cptr(), &mut value), value)
                }
            }
        }
        // ...
    };
    gen.into()
}
```

## Debugging

```bash
cargo install cargo-expand
# rustfmt for formatting output
rustup component add rustfmt-preview
# Pygments to colorize output
sudo pip install Pygments
# Output macro expansion
cargo expand
```

![]({{ "/assets/cargo_expand.png" | absolute_url }})

One thing to watch out for is the type of the `#XYZ` variables.  If you try to do something like:
```rust
let getopt_bool: String = prefix.clone() + "getopt_bool";
let gen = quote! {
    NngFail::succeed( #getopt_bool (args) )
};
```

`cargo check`:
```
error[E0618]: expected function, found `&'static str`
```

`cargo expand`:
```rust
// ...
fn getopt_bool(&self, option: NngOption) -> NngResult<bool> {
    unsafe {
        let mut value: bool = Default::default();
        NngFail::succeed("nng_getopt_bool"(self.socket,
                                            option.as_cptr(),
                                            &mut value), value)
    }
}
// ...
```

It generated `"nng_getopt_bool"(args)` instead of `nng_getopt_bool(args)`.  Give it a `String` and it will give you one right back!

## Resources

Resources and the reading order I'd recommend:  
1. [Rust Programming Language](https://doc.rust-lang.org/book/2018-edition/appendix-04-macros.html)- basic introduction
1. [The Reference](https://doc.rust-lang.org/reference/procedural-macros.html)- basic introduction
1. ["Creating an enum iterator using Macros 1.1"](https://cbreeden.github.io/Macros11/)- example of `derive` macro
1. ["Debugging Rust's new Custom Derive System"](https://quodlibetor.github.io/posts/debugging-rusts-new-custom-derive-system/)- overview of development/debugging
1. [`syn` examples](https://github.com/dtolnay/syn/tree/master/examples)
1. [syn](https://docs.rs/syn)- `syn` crate documentation.  This was needed to write `derive_nng_opts()` when working with the AST.
1. (Optional) [Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html)- covers `macro_rules!`
    - ["Macros in the AST"](https://danielkeep.github.io/tlborm/book/mbe-syn-macros-in-the-ast.html)

The last item, the LBRM, was actually one of the first things I read.  __Warning__: it's dense:  
![]({{ "/assets/dense.png" | absolute_url }})

And I talking about usage #1 and/or #3 here, not #2.  If nothing else, read the ["Macros in the AST"](https://danielkeep.github.io/tlborm/book/mbe-syn-macros-in-the-ast.html) section.  It's a good overview of macros in Rust and was helpful in coercing my mental model (string substitution) to reality (intermediary form used by compiler).

## );

I've barely scratched the surface here.  For one thing my slavish use of `syn::Ident::new(.., syn::export::Span::call_site())` is merely the first thing I got working.  The `syn` crate has a lot of stuff.

I'd like to revisit the [Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html) and take a shot at `macro_rules!`.