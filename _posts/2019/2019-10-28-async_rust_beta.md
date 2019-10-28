---
layout: post
title: Async Rust Beta- Quick Peek
tags:
- rust
- async
series: Async Rust
---

As announced on the [Rust Blog](https://blog.rust-lang.org/2019/09/30/Async-await-hits-beta.html) a few weeks ago, the [long awaited](https://areweasyncyet.rs) async-await syntax hit beta and is slated for release with 1.39 early November.  Take a look at [the Async Book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html) for an in-depth introduction.

In eager anticipation of async-await, we've been using futures 0.1 crate for some time.  But, we're long overdue to finally take a look at where things are headed.  The following is some loosely structured notes on the upcoming "stable" state of futures and async/await.

## Foot in the Door

`cargo new async` and in `Cargo.toml` add:
```toml
[dependencies]
futures = { version = "0.3.0-alpha", package = "futures-preview" }
```

Here using the "mostly final" [Futures 0.3](https://rust-lang-nursery.github.io/futures-rs/blog/2018/07/19/futures-0.3.0-alpha.1.html) crate.


As explained in [The Book](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html) there are three release channels: nightly, beta, and stable.  You may have used "nightly" before to access the latest and greatest features- this is where async/await previously lived.  "beta" is where a version goes before becoming the next "stable" release.

```bash
# Update all installed toolchains
rustup update
# List installed toolchains
rustup toolchain list
# Install beta toolchain
rustup install beta
```

There's a few ways to use the beta/nightly channels:
```bash
# List toolchain overrides
rustup override list
# Set default toolchain for directory
rustup override set beta # or "nightly"
# Now defaults to beta toolchain
cargo build

# Explicitly build using beta/nightly toolchain
cargo +beta build # or "+nightly"
```

In `src/main.rs`:
```rust
async fn hello_world() {
    println!("Hello world");
}

async fn start() {
    hello_world().await
}

fn main() {
    let future = start();
    futures::executor::block_on(future);
}
```

`cargo run` to greet the world.

Notice how `await` is post-fix instead of `await hello_world()` as found in many other languages.  The syntax [was heavily debated](https://internals.rust-lang.org/t/await-syntax-discussion-summary), but the rationale boils down to improving: method chaining, co-existance with the [`?` operator](https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html), and precedence rules.

A contrived example with a series of calls (some of which can fail):
```rust
let answer = can_fail().await?
    .some_func().await
    .member_can_fail().await?
    .get_answer()?;
```

You can't understand `async` without understanding Rust's [`Future` trait](https://docs.rs/futures-preview/0.3.0-alpha.19/futures/prelude/trait.Future.html).  Perhaps the first thing to learn about `Future` is they're lazy; nothing happens unless something "pumps" them- as `executor::block_on` does here.  Contrast this with [`std::thread::spawn`](https://doc.rust-lang.org/std/thread/fn.spawn.html) which creates a running thread.  If futures are polled, does that mean Rust async programming isn't event-driven à la epoll/kqueue?  Don't fret, a [`Waker`](https://docs.rs/futures-preview/0.3.0-alpha.19/futures/task/struct.Waker.html) can be used to signal the future is ready to be polled again.

## Error-Handling

We have test code like:
```rust
while !done.load(Ordering::Relaxed) {
    match block_on(ctx.receive()) {
        Ok(msg) => {
            let pipe = msg.get_pipe()?;
            let mut response = msg.dup()?;
            response.set_pipe(&pipe);
            block_on(ctx.send(response))?;
        }
        _ => panic!(),
    }
}
```

How we might re-write it:
```rust
let future = async {
    while !done.load(Ordering::Relaxed) {
        match ctx.receive().await {
            Ok(msg) => {
                let pipe = msg.get_pipe()?;
                let mut response = msg.dup()?;
                response.set_pipe(&pipe);
                ctx.send(response).await?;
            }
            _ => panic!(),
        }
    }
};

block_on(future);
```

Unfortunately, how the `?` operator works in async blocks (i.e. `async {}`) is [not defined](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md#-operator-and-control-flow-constructs-in-async-blocks), and async closures (i.e. `async || {}`) are [unstable](https://github.com/rust-lang/rust/issues/62290).

If we replace `?` with `.unwrap()` it compiles and runs.

## Heterogeneous Returns

Given:
```rust
broker_pull_ctx.receive().for_each(|msg| {
    if let Ok(msg) = msg {
        broker_push_ctx.send(msg).then(|msg| {
            // Do something with the message
            future::ready(())
        })
    } else {
        future::ready(())
    }
});
```

Sadness:
```rust
    |
144 |  /                 if let Ok(msg) = msg {
145 |  |                     broker_push_ctx.send(msg).then(|res| {
    |  |_____________________-
146 | ||                         res.unwrap().unwrap();
147 | ||                         future::ready(())
148 | ||                     })
    | ||______________________- expected because of this
149 |  |                 } else {
150 |  |                     future::ready(())
    |  |                     ^^^^^^^^^^^^^^^^^ expected struct `futures_util::future::then::Then`, found struct `futures_util::future::ready::Ready`
151 |  |                 }
    |  |_________________- if and else have incompatible types
    |
    = note: expected type `futures_util::future::then::Then<futures_channel::oneshot::Receiver<std::result::Result<(), runng::result::Error>>, futures_util::future::ready::Ready<_>, [closure@runng/tests/test_main.rs:145:52: 148:22]>`
               found type `futures_util::future::ready::Ready<_>`
```

Basically, `then()`- like many Rust combinators- [returns a distinct type](https://docs.rs/futures-preview/0.3.0-alpha.19/futures/future/trait.FutureExt.html#method.then) (`Then` in this case).

If you reach for a [trait object](https://doc.rust-lang.org/1.30.0/book/2018-edition/ch17-02-trait-objects.html) for type erasure via `-> Box<dyn Future<Output = ()>>` and wrap the returns in `Box::new()` you'll run into:
```rust
error[E0277]: the trait bound `dyn core::future::future::Future<Output = ()>: std::marker::Unpin` is not satisfied
   --> runng/tests/test_main.rs:155:58
    |
155 |             let fut = broker_pull_ctx.receive().unwrap().for_each(|msg| -> Box<dyn Future<Output = ()>> {
    |                                                          ^^^^^^^^ the trait `std::marker::Unpin` is not implemented for `dyn core::future::future::Future<Output = ()>`
    |
    = note: required because of the requirements on the impl of `core::future::future::Future` for `std::boxed::Box<dyn core::future::future::Future<Output = ()>>`
```

Lo, the 1.33 feature ["pinning"](https://doc.rust-lang.org/std/pin/).  Thankfully, the type-insanity that is `Pin<Box<dyn Future<Output = T>>>` is common enough that a [`future::BoxFuture<T>`](https://docs.rs/futures-preview/0.3.0-alpha.19/futures/future/type.BoxFuture.html) alias is provided:
```rust
let fut = broker_pull_ctx...for_each(|msg| -> future::BoxFuture<()> {
    if let Ok(msg) = msg {
        Box::pin(broker_push_ctx.send(msg).then(|_| { }))
    } else {
        Box::pin(future::ready(()))
    }
});
block_on(fut);
```

Alternatively, you can multiplex the return with something like [`future::Either`](https://docs.rs/futures-preview/0.3.0-alpha.19/futures/future/enum.Either.html):
```rust
let fut = broker_pull_ctx...for_each(|msg| {
    use futures::future::Either;
    if let Ok(msg) = msg {
        Either::Left(
            broker_push_ctx.send(msg).then(|_| { })
        )
    } else {
        Either::Right(future::ready(()))
    }
});
block_on(fut);
```

This avoids the boxing allocation, but it might become a bit gnarly if there's a large number of return types.

## `block_on()` != `.await`

Our initial, exploratory implementation made heavy use of [`wait()`](https://docs.rs/futures/0.1.29/futures/future/trait.Future.html#method.wait) found in futures 0.1.  To transition to async/await it's tempting to replace `wait()` with `block_on()`:

```rust
#[test]
fn block_on_panic() -> runng::Result<()> {
    let url = get_url();

    let mut rep_socket = protocol::Rep0::open()?;
    let mut rep_ctx = rep_socket.listen(&url)?.create_async()?;

    let fut = async {
        block_on(rep_ctx.receive()).unwrap();
    };
    block_on(fut);

    Ok(())
}
```

`cargo test block_on_panic` yields:
```rust
---- tests::reqrep_tests::block_on_panic stdout ----
thread 'tests::reqrep_tests::block_on_panic' panicked at 'cannot execute `LocalPool` executor from within another executor: EnterError', src/libcore/result.rs:1165:5
```

Note this isn't a compiler error, it's a runtime panic.  I haven't looked into the details of this, but the problem stems from the nested calls to `block_on()`.  It seems that if the inner future finishes immediately everything is fine, but not if it blocks.  However, it works as expected with await:

```rust
let fut = async {
    rep_ctx.receive().await.unwrap();
};
block_on(fut);
```

## Async Traits

Try:
```rust
trait AsyncTrait {
    async fn do_stuff();
}
```

Nope:
```rust
error[E0706]: trait fns cannot be declared `async`
 --> src/main.rs:5:5
  |
5 |     async fn do_stuff();
  |     ^^^^^^^^^^^^^^^^^^^^
```

How about:
```rust
trait AsyncTrait {
    fn do_stuff();
}

struct Type;

impl AsyncTrait for Type {
    async fn do_stuff() { }
}
```

Nope:
```rust
error[E0706]: trait fns cannot be declared `async`
  --> src/main.rs:11:5
   |
11 |     async fn do_stuff() { }
   |     ^^^^^^^^^^^^^^^^^^^^^^^
```

One work-around involves being explicit about the return types, and using an async block within the impl:
```rust
use futures::future::{self, BoxFuture, Future};

trait AsyncTrait {
    fn boxed_trait() -> Box<dyn Future<Output = ()>>;
    fn pinned_box() -> BoxFuture<'static, ()>;
}

impl<T> AsyncTrait for T {
    fn boxed_trait() -> Box<dyn Future<Output = ()>> {
        Box::new(async {
            // .await to your heart's content
        })
    }
    fn pinned_box() -> BoxFuture<'static, ()> {
        Box::pin(async {
            // .await to your heart's content
        })
    }
}
```
