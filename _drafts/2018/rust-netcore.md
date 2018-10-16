---
layout: post
title: Calling Rust library from .NET Core
tags:
- rust
- nng
- showdev
- csharp
- netcore
---

Continuing work on [Rust bindings to NNG]({% post_url /2018/2018-09-30-rust-ffi-ci %}).  I'm going to load Rust library as part of .Net Core generic host application.

Source code on [Github](https://github.com/jeikabu/runng).

### Most casts and serialization

Turning `Box<_>` into c_void:
```rust
ctx.as_mut() as *mut _ as *const ::std::os::raw::c_void
```

`Send` for pointer types
https://github.com/rust-lang/rust/issues/21709

### `msg_append()`

Nng defines numerous methods for working with messages.

[`nng_msg_append()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg_append.3.html) adds an array of bytes to the end of a message.  Bindgen:
```rust
pub fn nng_msg_append(
        arg1: *mut nng_msg,
        arg2: *const ::std::os::raw::c_void,
        arg3: usize,
    ) -> ::std::os::raw::c_int;
```

Almost immediately needed to find a way to pass `Vec<u8>` as a `*const c_void` for [this SO](https://stackoverflow.com/questions/31759582/assign-an-array-to-mut-c-void).  Coerces to `[u8]` slice and using [`as_ptr()`](https://doc.rust-lang.org/std/primitive.slice.html#method.as_ptr) like `self.header.as_ptr() as *const c_void`:
```rust
pub fn build(&self) -> NngResult<NngMsg> {
    let mut msg = NngMsg::new()?;
    //...
    let len = self.body.len();
    if len > 0 {
        msg.append(self.body.as_ptr() as *const c_void, len)?;
    }
    Ok(msg)
}
```

Where `NngMsg::append()` is a wrapper around `nng_msg_append()`:
```rust
pub fn append(&mut self, data: *const c_void, size: usize) -> NngReturn {
    unsafe {
        NngFail::from_i32(nng_msg_append(self.msg(), data, size))
    }
}
```

Rather than deal with all the native pointer types and calling into a native library for something as basic as contructing messages, I started a message builder in Rust.

[`nng_msg_append_u32()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg_append.3.html) adds a 32-bit integer in network-byte order to the end of a message.  Its binding:
```rust
pub fn nng_msg_append_u32(arg1: *mut nng_msg, arg2: u32) -> ::std::os::raw::c_int;
```

[This SO](https://stackoverflow.com/questions/29445026/converting-number-primitives-i32-f64-etc-to-byte-representations) covers different ways of turning `u32` into bytes, including using [`byteorder`](https://crates.io/crates/byteorder) crate to convert to [network-byte order (Big-endian)](https://en.wikipedia.org/wiki/Endianness#Networking):

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

### Subscribing to a topic

[Subscribing to a pub/sub topic](https://nanomsg.github.io/nng/man/v1.0.0/nng_sub.7.html) involves C/C++ similar to:
```c
nng_setopt(socket, NNG_OPT_SUB_SUBSCRIBE, (const void *)data, (size_t)size);
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

My subscribe method is mostly dealing with the types:
```rust
pub fn subscribe(&self, topic: &[u8]) -> NngReturn {
    unsafe {
        if let Some(ref aio) = self.ctx.aio {
            // Rust u8 array to const char*
            let opt = NNG_OPT_SUB_SUBSCRIBE.as_ptr() as *const ::std::os::raw::c_char;

            // Rust slice to const void* and size_t
            let topic_ptr = topic.as_ptr() as *const ::std::os::raw::c_void;
            let topic_size = std::mem::size_of_val(topic);

            let res = nng_setopt(aio.nng_socket(), opt, topic_ptr, topic_size);
```

Need [`std::mem::size_of_val()`](https://doc.rust-lang.org/std/mem/fn.size_of_val.html) to get size of `[u8]` slice.


## Callback from Native Code

`nng_aio_alloc()` takes a pointer to a method that is called when asynchronous IO operations complete:
```rust
pub fn nng_aio_alloc(
        arg1: *mut *mut nng_aio,
        arg2: ::std::option::Option<unsafe extern "C" fn(arg1: *mut ::std::os::raw::c_void)>,
        arg3: *mut ::std::os::raw::c_void,
    ) -> ::std::os::raw::c_int;
```

https://doc.rust-lang.org/stable/book/first-edition/ffi.html#callbacks-from-c-code-to-rust-functions


Mutible reference borrows:
```rust
extern fn pull_callback(arg : AioCallbackArg) {
    unsafe {
        let ctx = &mut *(arg as *mut AsyncPullContext);
        
        println!("callback Subscribe:{:?}", ctx.state);
        match ctx.state {
            PullState::Ready => panic!(),
            PullState::Receiving => {
                let aio = ctx.aio.as_ref().map(|aio| aio.aio());
                if let Some(aio) = aio {
                    let res = NngFail::from_i32(nng_aio_result(aio));
```

## Futures

In [our C# wrapper nng.NETCore](https://github.com/subor/nng.NETCore), most nng send/receive operations return `Task<>`.  In Rust, the [futures crate](https://crates.io/crates/futures) seems to be the best way to provide a similar interface.  Reading:

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

The error message is pretty clear: can't use `impl Trait` outside free-standing functions.  Seeing as how `impl Trait` was a [major feature of 1.26](https://blog.rust-lang.org/2018/05/10/Rust-1.26.html) this seemed odd.  Turns out [the reason is non-trivial](https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md#limitation-to-freeinherent-functions).

All the examples involve [simple ok](https://docs.rs/futures/0.1.25/futures/future/fn.ok.html), but we want something more like C#'s [TaskCompletionSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource-1) where we can "signal" the future from the our callback.

[futures::sync](https://docs.rs/futures/0.1.25/futures/sync/index.html) contains `mpsc` and `oneshot` channels.

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

When the async operation completes, in callback (heavily edited for clarity):
```rust
extern fn publish_callback(arg : AioCallbackArg) {
    let ctx = &mut *(arg as *mut AsyncPublishContext);
    //...
    if let Some(ref mut aio) = ctx.aio {
        let res = NngFail::from_i32(nng_aio_result(aio.aio()));
        //...
        let sender: oneshot::Sender<MsgFutureType> = ctx.sender.take().unwrap();
        sender.send(res).unwrap();
    }
    //...
}
```


## C# Interop

https://dev.to/living_syn/calling-rust-from-c-6hk

Defaults to producing static libraries, to generate a dynamic/shared library to `Cargo.toml` add:
```toml
[lib]
crate-type = ["dylib"]
```

This will produce a `.dylib` on OSX, `.so` on Linux, `.dll` on Windows, etc.  Also see the [cargo docs](https://doc.rust-lang.org/cargo/reference/manifest.html#building-dynamic-or-static-libraries).

In `lib.rs`:
```rust
#[no_mangle]
pub extern fn start() -> i32 {
    println!("Start!");
    0
}
```

Run `cargo build`.

For immediate satisfaction I copied it to my .net output folder, but I'll need to look into [loading it as a native assembly]({% post_url /2018/2018-09-09-native-assembly %}).

In the C#:
```csharp
[DllImport("rust_input")]
static extern int start();
```


# Specialization

https://play.rust-lang.org/?version=nightly&mode=debug&edition=2015
https://github.com/rust-lang/rust/issues/31844
https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md
https://stackoverflow.com/questions/34471212/how-to-implement-specialized-versions-of-a-generic-function
