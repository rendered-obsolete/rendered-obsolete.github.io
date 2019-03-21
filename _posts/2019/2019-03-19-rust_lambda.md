---
layout: post
title: Rust on Lambda
tags:
- rust
- aws
- serverless
---

[We're excited]({% post_url /2018/2018-11-30-aws-lambda-rust %}) about [Amazon's announcement of Rust Runtime for AWS Lambda](https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/).  We finally got around to looking into this.

## Rust 2018

Since the announcement the end of November, Rust 1.31 and [Rust 2018](https://doc.rust-lang.org/nightly/edition-guide/rust-2018/index.html) as well as `lambda_runtime = 0.2` have been released.

Changes to the module system mean the previous example can be simplified.  First, the original code:
```rust
#[macro_use]
extern crate lambda_runtime as lambda;
use lambda::error::HandlerError;
#[macro_use]
extern crate serde_derive;
```

Now, to rename the `lambda_runtime` crate, in `Cargo.toml`:
```toml
lambda = {version = "0.2", package = "lambda_runtime"}
```
And then in the `rs`:
```rust
use lambda::error::HandlerError;
use serde::{Serialize, Deserialize};
```

Check github for a complete version of the [updated example](https://github.com/awslabs/aws-lambda-rust-runtime/blob/master/lambda-runtime/examples/basic.rs).

## Setup

On Linux you can just:
```bash
rustup target add x86_64-unknown-linux-musl # First time only
cargo build --release --target x86_64-unknown-linux-musl
```
On OSX it will fail with:
```
Internal error occurred: Failed to find tool. Is `musl-gcc` installed?
```

Using musl is more involved on macOS (as always).  It's covered in the original AWS blog as well as [this post](https://chr4.org/blog/2017/03/15/cross-compile-and-link-a-static-binary-on-macos-for-linux-with-cargo-and-rust/).  The gist is using [this brew script](https://github.com/FiloSottile/homebrew-musl-cross) to install [musl-cross-make](https://github.com/richfelker/musl-cross-make):
```bash
brew install FiloSottile/musl-cross/musl-cross
```

That doesn't create `musl-gcc`, in the AWS blog he creates a soft link:
```
ln -s /usr/local/bin/x86_64-linux-musl-gcc /usr/local/bin/musl-gcc
```

musl-gcc is a script created by the [musl install target](http://git.musl-libc.org/cgit/musl/tree/Makefile?h=v1.1.21&id=1691b23955590d1eb66a11158fdd91c86337e886#n70).  The [issue he links](https://github.com/alexcrichton/cc-rs/issues/82) alludes to the solution, and it's more explicit in [this issue](https://github.com/zonyitoo/context-rs/issues/31):
```
CROSS_COMPILE=x86_64-linux-musl- cargo build --release --target x86_64-unknown-linux-musl
```

You also need to a create a `.cargo/config` file (in the root of the Rust project or `~/`) containing:
```toml
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

Otherwise cargo will fail with:
```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" "-Wl,--as-needed" "-Wl,-z,noexecstack" "-Wl,--eh-frame-hdr" "-m64" "-nostdlib"
<SNIP>
  = note: clang: warning: argument unused during compilation: '-nopie' [-Wunused-command-line-argument]
          ld: unknown option: --as-needed
          clang: error: linker command failed with exit code 1 (use -v to see invocation)
```


## AWS


### Lambda

You can mostly follow along with the original example, we'll just clarify a few things here.

1. For __Runtime__ make sure to select "Use custom runtime in function code or layer".

1. In the __Function code__ panel, from "Code entry type" drop-down select "Upload a .zip file".  Click orange "Save" button in upper-right of page:  
![](/assets/aws_lambda_upload.png)

1. Click __Test__ button next to it, put something for "Event name" and add:
    ```json
    {
      "firstName": "Rustacean"
    }
    ```
1. Click __Create__ button.  And back at the main screen press __Test__ again.
1. The `CustomOutput{ message: ... }` is at the top not in "log output":  
![](/assets/aws_lambda_result.png)

There's a few other interesting panels:
- __Environment variables__: could be perfect for `RUST_BACKTRACE` or maybe env_logger
- __Basic settings__: configure memory allowance and timeout

From [AWS Lambda Limits](https://docs.aws.amazon.com/lambda/latest/dg/limits.html):

| Category | Limit
|-|-|
| Memory | 128 MB to 3008 MB
| Function timeout | 90 seconds (15 minutes)
| Package size | 50MB zipped, 250MB unzipped

From our log output we see:
```
Memory Size: 128 MB	Max Memory Used: 40 MB
```

If we look at our package:
```
-rw-r--r--  1 jake  staff  1922463 Feb 15 12:50 rust.zip
-rwxr-xr-x  2 jake  staff  5908520 Feb 15 12:47 target/x86_64-unknown-linux-musl/release/bootstrap
```

2 MB compressed, 6 MB uncompressed; we're using less than half the minimum memory and <5% of the package size.  Rust was born for this stuff!

### Region Selection

If you happen to operate _between_ AWS regions, you might put more thought into region selection than you would otherwise.  We used to do informal performance tests, but recently came across [this "speedtest"](https://cloudharmony.com/speedtest-for-aws):

![](/assets/speedtest_aws_shanghai_china_telecom.png)

You can sort the columns and geek-out over the box-plots and generated statistics.

If you're in East Asia, you probably could have guessed Europe and the Eastern seaboard of North America aren't worth considering.  But between `ap-northeast-1` (Tokyo), `ap-northeast-2` (Seoul), and `ap-southeast-1` (Singapore)?  Less obvious.

All this begs the question, "why AWS, why not a regional provider?"  As it so happens we've got an [AliYun/AliCloud](https://www.alibabacloud.com/) account and its offerings are "oddly" similar to AWS including a Lambda-like equivalent.  Problem is, it lacks the derth of runtime options:  

![](/assets/aliyun_lambda_runtimes.png)

### Lambda@Edge

"Lambda@Edge" is the hip, marketing-savvy name given to the combination of Lambda and CloudFront (boring old CDN).

It has [even stricter limits](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html#limits-lambda-at-edge) than vanilla Lambda; limiting the size of payloads and rate of requests.

1. Make sure you're in "N. Virginia" region:  
![](/assets/aws_region.png)
1. __Create Function__
1. Under __Permissions__, for "Execution role" pick "Create a new role from AWS policy templates", and from "Policy templates" pick __Basic Lambda@Edge permissions__.  Again __Create function__
1. From __Configuration__ tab, in "Designer" select __CloudFront__:  
![](/assets/aws_lambda_cloudfront.png)
1. In "Configure triggers" panel click __Deploy to Lambda@Edge__ button
1. For "CloudFront event" you pick [one of four events](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-cloudfront-trigger-events.html) that trigger your function (note that the limits are different depending on the event)
1. Fill the rest of the stuff in and __Deploy__...

Fail:
```
Correct the errors below and try again.
Runtime must be one of: Node.js 6.10, Node.js 8.10.
Your function's execution role must be assumable by the edgelambda.amazonaws.com service principal.
```

Sure enough, in [Requirements and Restrictions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html#lambda-requirements-lambda-function-configuration) you have to use "nodejs".

_Words cannot convey my sadness_.

Seems that there's some interest in using runtimes other than nodejs.  Here's an issue in a [pre-cursor to aws-lambda-rust-runtime](https://github.com/srijs/rust-aws-lambda/issues/28) and another for [golang](https://github.com/aws/aws-lambda-go/issues/52).  Went ahead and [prodded the aws-lambda-rust-runtime project](https://github.com/awslabs/aws-lambda-rust-runtime/issues/88), but was promptly rebuffed.  Fine, fine... "loose lips sink ships" and all that.

We'll be back...