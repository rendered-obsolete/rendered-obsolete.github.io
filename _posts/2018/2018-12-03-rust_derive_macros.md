---
layout: post
title: Rust Derive Macros
tags:
- rust
- macros
- metaprogramming
series: Rust Macros
---

Rust has macros similar in purpose to C++'s macros/templates in that it's code that runs at compile time that itself writes code (meta-programming).  However, the similarities pretty much end there.  There are two types:

- Declarative macros (aka "macros by example" or `macro_rules!` macros)
- Procedural macros.  Which come in 3 varieties:
    - Function-like: e.g. `custom!()`
    - Derive mode: e.g. `#[derive(CustomMode)]` (applied to `struct` and `enum`)
    - Attribute: e.g. `#[CustomAttribute]` (applied to almost anything)

This is a quick primer on __derive__ mode procedural macros.

## Use Case: FFI to NNG

Quick summary of the problem we're trying to solve.

### Problem

NNG has a straight-forward C API.  Instead of the function overloading you would find in C++ (or many other languages), there's a collection of functions for each type.  The `nng_socket` type has __get__ and __set__ option functions (about a dozen in total):
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

Wrapping the bindgen generated functions in [our runng-sys crate](https://crates.io/crates/runng-sys) is easy:
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

Here we're already using helpers to deal with the boiler-plate:  
- `NngFail::succeed()` turns the C return value into one for Rust (`int -> Result<T>`)
- `.as_cptr()` turns an `enum` (with a `&'static [u8]`) into a C `const char *` naming the option (`NngOption -> *const i8`)

We started to replicate this mass of code for `nng_listener`, `nng_dialer`, et al.  But let's be honest, copy-pasting code is one of the most dubious [code smells](https://en.wikipedia.org/wiki/Code_smell).

### Solution

Using Rust's derive mode macros, we'd like to write something like:
```rust
#[derive(NngGetOpts,NngSetOpts)]
#[prefix = "nng_"]
#[nng_member = "socket"]
pub struct NngSocket {
    socket: nng_socket,
}
```

Which would substitute the `prefix` and `nng_member` values into `#PREFIXgetopt_bool(self.#NNG_MEMBER, ..` to form `nng_getopt_bool(self.socket, ..`.  Effectively source code string substitution with an end result similar to:
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

## Derive Mode Procedural Macro

Start a new library module with `cargo new --lib runng_derive` and in `Cargo.toml`:
```toml
[package]
name = "runng_derive"
# ...

[lib]
# 1
proc-macro = true

[dependencies]
# 2
syn = "0.15"
quote = "0.6"
```

Notes:
1. `proc-macro = true` is needed to avoid:

        error: the `#[proc_macro_derive]` attribute is only usable with crates of the `proc-macro` crate type

1. [syn](https://crates.io/crates/syn) and [quote](https://crates.io/crates/quote) crates do all the real work for us

In `lib.rs`:
```rust
#![recursion_limit="128"] // 1

extern crate proc_macro;
extern crate syn;
#[macro_use] // 2
extern crate quote;

use proc_macro::TokenStream;
use syn::{
    Lit,
    Meta,
    MetaNameValue,
};

// 3
#[proc_macro_derive(NngGetOpts, attributes(prefix, nng_member))]
pub fn derive_nng_get_opts(tokens: TokenStream) -> TokenStream {
    derive_nng_opts(tokens, gen_get_impl)
}
```

Notes: 
1. `recursion_limit` was recommended by the compiler after failing with:  

        error: recursion limit reached while expanding the macro `stringify

1. Bring `quote` crate's macros into scope (NB: [not required in Rust 2018](https://rust-lang-nursery.github.io/edition-guide/rust-2018/macros/macro-changes.html))
1. `#[proc_macro_derive]` defines our `#[derive(..)]` macro.  Within that, `attributes(..)` defines our ["derive mode helper attributes"](https://doc.rust-lang.org/reference/procedural-macros.html#derive-mode-helper-attributes) (i.e. `#[prefix]` and `#[nng_member]`).

The input to this function is the stream of tokens of whatever code the attribute is attached to.  The output is the new code we want to generate.

First extract our `#[prefix ..]` and `#[nng_member ..]` attributes:
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
            // Match `#[ident = lit]` attributes.  Match guard makes it `#[prefix = lit]`
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
    
    // Generate the `impl`
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

The [`quote` macro](https://docs.rs/quote/0.6.10/quote/macro.quote.html) does the work of taking the source code we provide, substituting in values prefixed with `#`, and then generating a `TokenStream` to return from our macro.

## Debugging

[__cargo-expand__](https://github.com/dtolnay/cargo-expand) (by the author of the `syn` crate) prints out the result of macro expansion and is handy for debugging:  
```bash
# Nightly toolchain must be installed (it doesn't need to be default)
rustup toolchain install nightly
# Install `cargo expand`
cargo install cargo-expand
# rustfmt for formatting output
rustup component add rustfmt-preview
# Pygments to colorize output
sudo pip install Pygments
# Output macro expansion
cargo expand
```

![]({{ "/assets/cargo_expand.png" | absolute_url }})

### Example

One thing to watch out for is the type of the `#XYZ` variables.  Without the call to `syn::Ident::new()`:
```rust
    let getopt_bool = prefix.clone() + "getopt_bool";
    //...
    fn getopt_bool(&self, option: NngOption) -> NngResult<bool> {
        unsafe {
            let mut value: bool = Default::default();
            NngFail::succeed( #getopt_bool ( /* ... */ )
        }
    }
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

It generated `"nng_getopt_bool"(...)` when we need `nng_getopt_bool(...)`.  Give it a `String` and it will give you one right back!

`cargo expand` dumps the output for the entire module which might be more than you want.  It can be limited to a single test with `cargo expand --test name_of_the_test` and there's [some other useful options](https://github.com/dtolnay/cargo-expand#options).

## More Helpful Helper

Explicitly naming the field with `#[nng_member = "socket"]` is brittle.  What we really want is to extract the name of the field based on the placement of the attribute:  

```rust
#[derive(NngGetOpts,NngSetOpts)]
#[prefix = "nng_"]
pub struct NngSocket {
    #[nng_member]
    socket: nng_socket,
}
```

It's pretty easy to walk through the hierarchy of `enums` and `struct`s representing the source code:
```rust
fn get_nng_member(ast: &syn::DeriveInput) -> Option<syn::Ident> {
    match ast.data {
        syn::Data::Struct(ref data_struct) => {
            match data_struct.fields {
                // Structure with named fields (as opposed to tuple-like struct or unit struct)
                // E.g. struct Point { x: f64, y: f64 }
                syn::Fields::Named(ref fields_named) => {
                    // Iterate over the fields: `x`, `y`, ..
                    for field in fields_named.named.iter() {
                        // Get attributes `#[..]` on each field
                        for attr in field.attrs.iter() {
                            // Parse the attribute
                            let meta = attr.parse_meta().unwrap();
                            match meta {
                                // Matches attribute with single word like `#[nng_member]` (as opposed to `#[derive(NngGetOps)]` or `#[nng_member = "socket"]`)
                                Meta::Word(ref ident) if ident == "nng_member" =>
                                    // Return name of field with `#[nng_member]` attribute
                                    return field.ident.clone(),
                                _ => (),
                            }
                        }
                    }
                },
                _ => (),
            }
        },
        _ => panic!("Must be a struct"),
    }
    
    None
}
```

This is a pretty basic implementation; there's no error handling, doesn't support `#[nng_member]` on a nested field, etc.  But, since this macro is just used internally to our crate for educational purposes it's probably ok for now.

All this is pretty similar to [a common use of discriminated unions in F#](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions#using-discriminated-unions-for-tree-data-structures) (and probably every other language with functional tendencies).  Supporting more features and robust handling will probably turn into an exercise in recursive `match`ing.


## Resources

For further information on macros, and the reading order I would recommend:  

1. [Rust Programming Language](https://doc.rust-lang.org/book/ch19-06-macros.html)- basic introduction
1. [The Reference](https://doc.rust-lang.org/reference/macros.html)- basic introduction
1. ["Creating an enum iterator using Macros 1.1"](https://cbreeden.github.io/Macros11/)- example of `derive` macro
1. ["Debugging Rust's new Custom Derive System"](https://quodlibetor.github.io/posts/debugging-rusts-new-custom-derive-system/)- overview of development/debugging
1. [`syn` examples](https://github.com/dtolnay/syn/tree/master/examples)
1. [syn](https://docs.rs/syn)- `syn` crate documentation.  This is needed to write `derive_nng_opts()` when working with the AST.
1. (Optional) [Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html)- covers `macro_rules!`
    - ["Macros in the AST"](https://danielkeep.github.io/tlborm/book/mbe-syn-macros-in-the-ast.html)

The last item, the LBRM, was actually one of the first things I read but probably the worst place to start.  If nothing else, read the ["Macros in the AST"](https://danielkeep.github.io/tlborm/book/mbe-syn-macros-in-the-ast.html) section.  It's a good overview of macros in Rust and was helpful in coercing my mental model (string substitution) to reality (intermediary form used by compiler).

## ); // End of Story

We've barely scratched the surface here.  For one thing my slavish use of `syn::Ident::new(.., syn::export::Span::call_site())` warrants investigation- it's merely the first thing that worked.  The `syn` crate has a lot of stuff.

I'd like to revisit the [Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html) and take a shot at `macro_rules!`.