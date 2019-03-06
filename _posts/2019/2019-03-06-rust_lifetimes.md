---
layout: post
title: Rust Lifetimes for Safer FFI
tags:
- rust
- showdev
- ffi
---

Lifetimes are one of Rust's marquee features and pivotal to its safety gurantees.  My understanding of them felt largely academic until I found a situation doing FFI to native code that warranted further investigation.

## Lifetimes 101

If you're not familiar with Rust lifetimes, here's the basic concept; they assist the compiler ensure references don't exist longer than the thing they reference:

```rust
// Type contains a reference to an integer with a lifetime of 'a
struct MyStruct<'a> (&'a i32);

 #[test]
 fn lifetime() {
     let inner = 42;
     // Create an instance of MyStruct with a reference to `inner` and a lifetime of 'a
     let outer = MyStruct(&inner);
 }
```

Here `outer` should not be allowed to outlive `inner` otherwise you would have a "dangling reference"- a pointer to something that no longer exists.  In languages with a garbage collector this is generally a non-issue; `inner` could be kept in the heap for as long as needed.  In languages like C/C++ care must be taken (aided by the compiler) to avoid problems like returning a reference/pointer to the stack, "use after free", and other errors.

## Use Case

[I've been working on]({% post_url /2018/2018-10-17-rust-ffi-nng %}) a Rust wrapper for a C library, [nng](https://github.com/nanomsg/nng).  A few months ago I was wrangling runtime statistics.

[`nng_stats_get()`](https://nanomsg.github.io/nng/man/v1.1.0/nng_stats_get.3) returns a snapshot of runtime statistics as a tree which can be traversed with [`nng_stat_child()`](https://nanomsg.github.io/nng/man/v1.1.0/nng_stat_child.3) and [`nng_stat_next()`](https://nanomsg.github.io/nng/man/v1.1.0/nng_stat_next.3).  When no longer needed, [`nng_stats_free()`](https://nanomsg.github.io/nng/man/v1.1.0/nng_stats_free.3) releases the memory associated with the snapshot invalidating the entire tree.

A simple Rust wrapper:
```rust
// Root of statistics tree
pub struct NngStatRoot {
    node: *mut nng_stat,
}

impl NngStatRoot {
    // Create a snapshot of statistics
    pub fn create() -> Option<NngStatRoot> {
        unsafe {
            // Get snapshot as pointer to memory allocated by C library
            let mut node: *mut nng_stat = std::ptr::null_mut();
            let res = nng_stats_get(&mut node);
            if res == 0 {
                Some(NngStatRoot { node })
            } else {
                None
            }
        }
    }
    // Get first "child" node of tree
    pub fn child(&self) -> Option<NngStatChild> {
        unsafe {
            let node = nng_stat_child(self.node);
            NngStatChild::new(node)
        }
    }
}

// When root goes out of scope free the memory
impl Drop for NngStatRoot {
    fn drop(&mut self) {
        unsafe {
            nng_stats_free(self.node)
        }
    }
}

// A "child"; any non-root node of tree
pub struct NngStatChild {
    node: *mut nng_stat,
}

impl NngStatChild {
    // Create a child
    pub fn new(node: *mut nng_stat) -> Option<NngStatChild> {
        if node.is_null() {
            None
        } else {
            Some(NngStatChild { node })
        }
    }
    // Get sibling of this node
    pub fn next(&self) -> Option<NngStatChild> {
        unsafe {
            let node = nng_stat_next(self.node);
            NngStatChild::new(node)
        }
    }
    // Get first "child" of this node
    pub fn child(&self) -> Option<NngStatChild> {
        unsafe {
            let node = nng_stat_child(self.node);
            NngStatChild::new(node)
        }
    }
}
```

This can be used as follows:
```rust
fn stats() {
    let root = NngStatRoot::create().unwrap();
    if let Some(child) = root.child() {
        if let Some(sibling) = child.next() {
            // Do something
        }
    }
} // root dropped here calling nng_stats_free()
```

Unfortunately, the following also "works" (it compiles and maybe runs), but is results in "undefined behavior":
```rust
fn naughty_code() {
    let mut naughty_child: Option<_> = None;
    {
        let root = NngStatRoot::create().unwrap();
        naughty_child = root.child();
    } // root dropped here calling nng_stats_free()

    if let Some(child) = naughty_child {
        if let Some(naughty_sibling) = child.next() {
            debug!("Oh no!");
        }
    }
}
```

The problem is `naughty_child` allows a pointer into the statistics snapshot to outlive `root` and be accessed after `nng_stats_free()` is called.

## Solution Using Lifetimes

I was pretty sure this was a job for lifetimes.

Once you give a struct a lifetime it "infects" everything it touches:
```rust
pub struct NngStatRoot<'root> {
    node: *mut nng_stat,
}
impl<'root> NngStatRoot<'root> {
    // Create a snapshot of statistics
    pub fn create() -> Option<NngStatRoot<'root>> {
        //...
    }
//...
```

In particular, note `impl<'root>`.  Without that you get:
```rust
error[E0261]: use of undeclared lifetime name `'root`
  --> runng/tests/tests/stream_tests.rs:77:18
   |
77 | impl NngStatRoot<'root> {
   |                  ^^^^^ undeclared lifetime
```

After applying the lifetime everywhere you'll evetually get:
```rust
error[E0392]: parameter `'root` is never used
  --> runng/tests/tests/stream_tests.rs:73:24
   |
73 | pub struct NngStatRoot<'root> {
   |                        ^^^^^ unused type parameter
   |
   = help: consider removing `'root` or using a marker such as `std::marker::PhantomData`
```

Lifetime `'root` is unused.  It cannot be applied to the pointer:
```rust
pub struct NngStatRoot<'root> {
    // NB: doesn't compile
    node: *'root mut nng_stat,
}
```

Lifetimes don't go on pointers, only references (`&`):
```rust
pub struct NngStatRoot<'root> {
    node: &'root mut nng_stat,
}
```

Switching to a reference has two problems:
1. Requires __lots__ of casting because the native methods take pointers (`*`)
1. Need to be more careful with `mut` on instances of the struct

The helpful compiler message alludes to another solution, [`std::marker::PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), which allows our struct to "act like" it owns a reference:

```rust
pub struct NngStatRoot<'root> {
    node: *mut nng_stat,
    _phantom: marker::PhantomData<&'root nng_stat>,
}

impl<'root> NngStatRoot<'root> {
    pub fn create() -> Result<NngStatRoot<'root>> {
        unsafe {
            let mut node: *mut nng_stat = std::ptr::null_mut();
            let res = nng_stats_get(&mut node);
            if res == 0 {
                Some(NngStatRoot {
                    node,
                    // "Initialize" the phantom
                    _phantom: marker::PhantomData,
                })
            } else {
                None
            }
        }
    }
}
```

What's cool is `PhantomData` is a [Zero-Sized Type](https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts); it has no runtime cost (neither CPU nor memory), it exists only at compile-time:
```rust
pub struct Phantom<'root> {
    _phantom: marker::PhantomData<&'root nng_stat>,
}

#[test]
fn check_size() {
    assert_eq!(0, std::mem::size_of::<Phantom>());
}
```

One place that's warrants special attention is our `next()` method:
```rust
impl<'root> NngStatChild<'root> {
    //...

    // NB: The explicit lifetime on the return value is key!
    pub fn next(&self) -> Option<NngStatChild<'root>> {
        unsafe {
            let node = nng_stat_next(self.node);
            NngStatChild::new(node)
        }
    }
}
```

We need an explicit lifetime here because without it the [lifetime ellision](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-elision) rules would assign the same lifetime as `&self`.  That implies that the lifetimes of the siblings are somehow related, but all that matters is the lifetime of the root.

Let's revisit our naughty code:
```rust
fn naughty_code() {
    let mut naughty_child: Option<_> = None;
    {
        let root = NngStatRoot::create().unwrap();
        naughty_child = root.child();
    } // root dropped here calling nng_stats_free()

    if let Some(child) = naughty_child {
        if let Some(naughty_sibling) = child.next() {
            debug!("Oh no!");
        }
    }
}
```

Now when we build it the compiler lets us know we did something bad:
```rust
error[E0597]: `root` does not live long enough
  --> runng/tests/tests/stats_tests.rs:37:25
   |
37 |         naughty_child = root.child();
   |                         ^^^^ borrowed value does not live long enough
38 |     } // root dropped here calling nng_stats_free()
   |     - `root` dropped here while still borrowed
39 | 
40 |     if let Some(child) = naughty_child {
   |                          ------------- borrow later used here
```

Crisis averted, thanks compiler!

## Fin

If you read this far Rust might be your cup of tea and you should give it a look- if you haven't already.

Full source is on [github](https://github.com/jeikabu/runng/blob/master/runng/src/stats.rs).

Further reading:
- The [Rust Programming Language](https://doc.rust-lang.org/book/index.html) ("the book") devotes two chapters to lifetimes:
    - [Chapter 10](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) "Validating References with Lifetimes"
    - [Chapter 19](https://doc.rust-lang.org/book/ch19-02-advanced-lifetimes.html) "Advanced Lifetimes"
- The Rustonomicon has a great [section on PhantomData](https://doc.rust-lang.org/nomicon/phantom-data.html) that I wish I would have found before I started working on this

