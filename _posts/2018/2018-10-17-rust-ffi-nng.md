---
layout: post
title: Rust FFI to NNG
tags:
- rust
- nng
- showdev
---

[Continuing work]({% post_url /2018/2018-09-30-rust-ffi-ci %}) on a [Rust](https://www.rust-lang.org/en-US/) library for [nng (nanomsg-next-gen)](https://github.com/nanomsg/nng).  Using [bindgen](https://github.com/rust-lang-nursery/rust-bindgen) to generate [FFI bindings](https://doc.rust-lang.org/book/ffi.html) to the C library and then a high-level wrapper over that.

This is mostly a brain-dump of problems I ran into while I'm still learning Rust; dealing with types, the [borrow checker](https://doc.rust-lang.org/book/references-and-borrowing.html), etc.

Source code on [Github](https://github.com/jeikabu/runng).

## Message Operations

Nng defines numerous methods for working with messages.

[`nng_msg_append()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg_append.3.html) adds an array of bytes to the end of a message.  Bindgen creates:
```rust
pub fn nng_msg_append(
        arg1: *mut nng_msg,
        arg2: *const ::std::os::raw::c_void,
        arg3: usize,
    ) -> ::std::os::raw::c_int;
```

We almost immediately want a way to pass `Vec<_>` as `*const c_void`.  [This SO](https://stackoverflow.com/questions/31759582/assign-an-array-to-mut-c-void) provides a solution; `Vec<u8>` coerces to `[u8]` slice and can then use [`as_ptr()`](https://doc.rust-lang.org/std/primitive.slice.html#method.as_ptr):
```rust
pub fn build(&self) -> NngResult<NngMsg> {
    let mut msg = NngMsg::new()?;
    //...
    let len = self.body.len();
    if len > 0 {
        let ptr = self.body.as_ptr() as *const c_void;
        unsafe {
            NngFail::from_i32(nng_msg_append(self.msg(), ptr, len))?;
        }
    }
    Ok(msg)
}
```

In order to provide semantics more natural to Rust (and avoid dealing with all the pointer types and calling into a C library for such basic operations), I started a message builder in Rust.  For example, [`nng_msg_append_u32()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg_append.3.html) adds a 32-bit integer in __network-byte order__ to the end of a message.  Its binding:
```rust
pub fn nng_msg_append_u32(arg1: *mut nng_msg, arg2: u32) -> ::std::os::raw::c_int;
```

[This SO](https://stackoverflow.com/questions/29445026/converting-number-primitives-i32-f64-etc-to-byte-representations) covers different ways of turning `u32` into bytes, including using the [`byteorder`](https://crates.io/crates/byteorder) crate to convert to [network-byte order (Big-endian)](https://en.wikipedia.org/wiki/Endianness#Networking):

```rust
extern crate byteorder;

use self::byteorder::{BigEndian, WriteBytesExt};

pub struct MsgBuilder {
    header: Vec<u8>,
    body: Vec<u8>,
}

impl MsgBuilder {
    pub fn append_u32(&mut self, data: u32) -> &mut Self {
        let mut bytes = [0u8; std::mem::size_of::<u32>()];
        bytes.as_mut().write_u32::<BigEndian>(data).unwrap();
        self.append_slice(&bytes)
    }
    pub fn append_slice(&mut self, data: &[u8]) -> &mut Self {
        self.body.extend_from_slice(data);
        self
    }
}
```

## Subscribing to a topic

[Subscribing to a pub/sub topic](https://nanomsg.github.io/nng/man/v1.0.0/nng_sub.7.html) requires the C/C++:
```c
nng_setopt(subscribe_socket, NNG_OPT_SUB_SUBSCRIBE, (const void *)topic_name, (size_t)topic_name_size);
```

[`nng_setopt()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_setopt.3.html) and `NNG_OPT_SUB_SUBSCRIBE` as created by bindgen:
```rust
pub fn nng_setopt(
        arg1: nng_socket,
        arg2: *const ::std::os::raw::c_char,
        arg3: *const ::std::os::raw::c_void,
        arg4: usize,
    ) -> ::std::os::raw::c_int;
//...
pub const NNG_OPT_SUB_SUBSCRIBE: &'static [u8; 14usize] = b"sub:subscribe\0";
```

Our subscribe wrapper method is mostly dealing with the types:
```rust
pub fn subscribe(&self, topic: &[u8]) -> NngReturn {
    unsafe {
        if let Some(ref aio) = self.ctx.aio {
            // Rust u8 array to C const char*
            let opt = NNG_OPT_SUB_SUBSCRIBE.as_ptr() as *const ::std::os::raw::c_char;

            // Rust u8 slice to C const void* and size_t
            let topic_ptr = topic.as_ptr() as *const ::std::os::raw::c_void;
            let topic_size = std::mem::size_of_val(topic);

            let res = nng_setopt(aio.nng_socket(), opt, topic_ptr, topic_size);
```

Need [`std::mem::size_of_val()`](https://doc.rust-lang.org/std/mem/fn.size_of_val.html) to get size of `[u8]` slice.


## Callback from Native Code

[`nng_aio_alloc()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_aio_alloc.3.html) allocates an asynchronous I/O handle:
```rust
pub fn nng_aio_alloc(
        arg1: *mut *mut nng_aio,
        arg2: ::std::option::Option<unsafe extern "C" fn(arg1: *mut ::std::os::raw::c_void)>,
        arg3: *mut ::std::os::raw::c_void,
    ) -> ::std::os::raw::c_int;
```

The second argument is a pointer to a method that is executed when an asynchronous I/O operation completes.  It is passed the last argument when called.

We need a Rust function that can be called from the C library.  There's [a blurb in the first edition](https://doc.rust-lang.org/stable/book/first-edition/ffi.html#callbacks-from-c-code-to-rust-functions) of _The Rust Programming Language_ ("The Book") on how to do this:

```rust
extern fn pull_callback(arg : *mut ::std::os::raw::c_void) {
    //...
}
```

We allocate an aio context and register the callback:
```rust
fn create_pull_aio(ctx: Box<AsyncPullContext>) {
    // Rust `Box<_>` into void*
    let ctx = ctx.as_mut() as *mut _ as *mut ::std::os::raw::c_void;

    let mut aio: *mut nng_aio = ptr::null_mut();
    let res = nng_aio_alloc(&mut aio, Some(pull_callback), ctx);
```

We turn a [boxed](https://doc.rust-lang.org/std/boxed/struct.Box.html) context into a `void*` that will be passed to our callback.  When an I/O operation completes and our callback is called, we get back our `&mut AsyncPullContext`:

```rust
extern fn pull_callback(arg : *mut ::std::os::raw::c_void) {
    unsafe {
        // Convert C void* to Rust `&mut AsyncPullContext`
        let ctx = &mut *(arg as *mut AsyncPullContext);

        match ctx.state {
            PullState::Ready => panic!(),
            PullState::Receiving => {
                let aio = ctx.aio.as_ref().map(|aio| aio.aio()); // -> Option<nng_aio>
                if let Some(aio) = aio {
                    // Check if the async I/O succeeded or failed
                    let res = NngFail::from_i32(nng_aio_result(aio));
                    //...
                    ctx.start_receive();
```

The line extracting `Option<nng_aio>` warrants explanation.  In other places I use the more typical:
```rust
if let Some(ref mut aio) = ctx.aio {
```

But I can't do that here:
```
error[E0499]: cannot borrow `*ctx` as mutable more than once at a time
   --> runng/src/protocol/pull.rs:113:37
    |
101 |                 if let Some(ref mut aio) = ctx.aio
    |                             ----------- first mutable borrow occurs here
...
113 |                                     ctx.start_receive();
    |                                     ^^^ second mutable borrow occurs here
...
128 |                 }
    |                 - first borrow ends here
```

Where:
```rust
impl AsyncPullContext {
    fn start_receive(&mut self) {
    //...
```

Basically, I can't unwrap the `Option<_>` field as a mutable reference and in the same scope call a method that also borrows a mutable reference to the struct.  Fine, try removing `mut`:
```
error[E0502]: cannot borrow `*ctx` as mutable because `ctx.aio.0` is also borrowed as immutable
   --> runng/src/protocol/pull.rs:113:37
    |
101 |                 if let Some(ref aio) = ctx.aio
    |                             ------- immutable borrow occurs here
...
113 |                                     ctx.start_receive();
    |                                     ^^^ mutable borrow occurs here
...
128 |                 }
    |                 - immutable borrow ends here
```

Right, can't have simultaneous immutable and mutable borrows.  The only thing that would work is multiple immutable borrows.

So, I use [`as_ref()`](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref) to get an `Option<&Rc<NngAio>>` then [`map()`](https://doc.rust-lang.org/std/option/enum.Option.html#method.map) to extract the `nng_aio` struct (which is copyable).

Technically, this isn't safe.  I'm abusing the fact that the C socket handles are `int`s (which copy).  But, if I were to start copying the handle around and using it different places things would break.  I'm going to look at restructering the code and/or moving `start_receive()`, but it wasn't immediately obvious how to do this.

## Futures

In [our C# wrapper nng.NETCore](https://github.com/subor/nng.NETCore), most nng send/receive operations return `Task<>`.  In Rust, the [futures crate](https://crates.io/crates/futures) seems to be the best way to provide a similar interface.  Background reading:

- http://aturon.github.io/blog/2016/08/11/futures/
- https://tokio.rs/docs/getting-started/futures/
- https://paulkernfeld.com/2018/01/20/future-by-example.html
- https://dev.to/mindflavor/rust-futures-an-uneducated-short-and-hopefully-not-boring-tutorial---part-1-3k3

Started with:
```rust
type MsgFuture = Future<Item=NngMsg, Error=()>;

pub trait AsyncReqRep {
    fn send(&mut self) -> impl MsgFuture;
}
```

And fails with:
```
error[E0562]: `impl Trait` not allowed outside of function and inherent method return types
```

The error message is pretty clear: can't use `impl Trait` outside free-standing functions.  Seeing as how `impl Trait` was a [major feature of 1.26](https://blog.rust-lang.org/2018/05/10/Rust-1.26.html) this seems odd.  Turns out [the reason is non-trivial](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md#limitation-to-freeinherent-functions).

All the examples involve [simply returning or using `ok`](https://docs.rs/futures/0.1.25/futures/future/fn.ok.html), but we want something more like C#'s [`TaskCompletionSource`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource-1) where we can "signal" the future from the asynchronous callback.  [futures::sync](https://docs.rs/futures/0.1.25/futures/sync/index.html) contains `mpsc` and `oneshot` channels:

```rust
impl AsyncRequest for AsyncRequestContext {
    fn send(&mut self, msg: NngMsg) -> MsgFuture {
        let (sender, receiver) = oneshot::channel::<MsgFutureType>();
        self.sender = Some(sender);
        //...
        receiver
    }
}
```

When the async operation completes, we signal the future from our callback (heavily edited for clarity):
```rust
extern fn publish_callback(arg : AioCallbackArg) {
    let ctx = &mut *(arg as *mut AsyncPublishContext);
    //...
    if let Some(ref mut aio) = ctx.aio {
        let res = NngFail::from_i32(nng_aio_result(aio.aio()));
        //...
        let sender = ctx.sender.take().unwrap(); // -> oneshot::Sender<MsgFutureType>
        sender.send(res).unwrap();
    }
    //...
}
```

## "Done"

Been enjoying Rust a lot more than when I first looked at it [in early 2016](https://rendered-obsolete.github.io/2016/03/21/rustlang-dynamic-library.html).  Still struggling a bit  designing with traits instead of inheritance, but it will become second nature soon enough.

Next up, we'll fold this into our .NET Core project and get them talking over nng.