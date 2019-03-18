---
layout: post
title: Rust on Lambda
tags:
- rust
- aws
- serverless
---

[AWS Rust lambda]({% post_url /2018/2018-11-30-aws-lambda-rust %})

https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/

https://serde.rs/derive.html
```rust
#[macro_use]
extern crate lambda_runtime as lambda;
use lambda::error::HandlerError;
```
```toml
lambda = {version = "0.2", package = "lambda_runtime"}
```
```rust
use lambda::error::HandlerError;
```

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

That doesn't create musl-gcc, in the AWS blog he creates a soft link:
```
ln -s /usr/local/bin/x86_64-linux-musl-gcc /usr/local/bin/musl-gcc
```

musl-gcc is a script created by the [musl install target](http://git.musl-libc.org/cgit/musl/tree/Makefile?h=v1.1.21&id=1691b23955590d1eb66a11158fdd91c86337e886#n70).  The [issue he links](https://github.com/alexcrichton/cc-rs/issues/82) alludes to the solution, and it's more explicit in [this issue](https://github.com/zonyitoo/context-rs/issues/31):
```
CROSS_COMPILE=x86_64-linux-musl- cargo build --release --target x86_64-unknown-linux-musl
```

You also need to a create a `.cargo/config` file (in the root of the Rust project) containing:
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

For __Runtime__ make sure to select "Use custom runtime in function code or layer".

"Function code" panel, "Code entry type" drop-down select "Upload a .zip file".  Click orange "Save" button in upper-right:  
![](/assets/aws_lambda_upload.png)

Click __Test__ button next to it, put something "Event name" and  add:
```json
{
  "firstName": "Rustacean"
}
```
__Create__ button.  And back on the screen press __Test__ again.
The `CustomOutput{ message: ... }` is in the top not in "log" output:  
![](/assets/aws_lambda_result.png)

"Environment variables" panel.
Perfect for `RUST_BACKTRACE` or maybe env_logger

"Basic settings" panel.

[AWS Lambda Limits](https://docs.aws.amazon.com/lambda/latest/dg/limits.html):
|||
|-|-
| Memory | 128 MB to 3008 MB
| Function timeout | 90 seconds (15 minutes)
| Package size | 50MB zipped, 250MB unzipped

```
Memory Size: 128 MB	Max Memory Used: 40 MB
```

```
-rw-r--r--  1 jake  staff  1922463 Feb 15 12:50 rust.zip
-rwxr-xr-x  2 jake  staff  5908520 Feb 15 12:47 target/x86_64-unknown-linux-musl/release/bootstrap
```

So, we're using less than half the minimum memory and <5% of the package size.  Rust was born for this stuff!

### Region Selection

https://cloudharmony.com/speedtest-for-aws

![](/assets/speedtest_aws_shanghai_china_telecom.png)

You can sort the columns and geek-out over the box-plots and generated statistics.

You probably could have guessed Europe and East coast of North America aren't worth considering.
But between `ap-northeast-1` (Tokyo), `ap-northeast-2` (Seoul), and `ap-southeast-1` (Singapore)?

All this begs the question, why AWS, why not a regional provider?  As it so happens we've got an AliYun account and it's oddly similar to AWS including a Lambda-like equivalent service.  Problem is, it lacks the derth of runtime options, 

![](/assets/aliyun_lambda_runtimes.png)

### Lambda@Edge

Lambda + Cloudfront

Even stricter limits:
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html#limits-lambda-at-edge

https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-cloudfront-trigger-events.html

![](/assets/aws_region.png)

![](/assets/aws_lambda_cloudfront.png)

"Basic Lambda@Edge permissions"

"Configure triggers" panel click "Deploy to Lambda@Edge" button.

```
Correct the errors below and try again.
Runtime must be one of: Node.js 6.10, Node.js 8.10.
Your function's execution role must be assumable by the edgelambda.amazonaws.com service principal.
```

In [Requirements and Restrictions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html#lambda-requirements-lambda-function-configuration).

_Words cannot convey my sadness_.

Seems that there's some interest in using runtimes other than nodejs.  Here's an issue in a [pre-cursor to aws-lambda-rust-runtime](https://github.com/srijs/rust-aws-lambda/issues/28) and another for [golang](https://github.com/aws/aws-lambda-go/issues/52).