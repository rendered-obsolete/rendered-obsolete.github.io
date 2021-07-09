---
layout: post
title: Cross-posting to Tumblr with AWS Lambda
tags:
- rust
- serverless
- productivity
- cli
- aws
- tumblr
- lambda
- rest
description: Extending my Rust CLI tool, BULLHORN, to cross-post to Tumblr.  OAuth authentication via AWS Lambda.
---

I really enjoyed [making a CLI tool to automate cross-posting blog posts to different places](https://rendered-obsolete.github.io/2021/06/19/bullhorn.html).  So, I started looking for another platform to add and Tumblr seemed like a good candidate.  Again, rough code is [on Github](https://github.com/jeikabu/cargo_bullhorn/).

## Tumblr API

The Tumblr API is [well documented](https://www.tumblr.com/docs/en/api/v2).  But before you get started you need to [register an "application"](https://www.tumblr.com/oauth/apps).  For "Default callback URL" put a valid URL, you can always edit this later (via __Account > Settings > Apps > ✎__).  This will get you an OAuth "consumer" key and secret (sometimes referred to as "client" credentials).

Some parts of the Tumblr API can be called with just the consumer key.  For example, to [retrieve published posts](https://www.tumblr.com/docs/en/api/v2#posts--retrieve-published-posts):

```powershell
curl -G https://api.tumblr.com/v2/blog/{blog-identifier}/posts?api_key={oauth_consumer_key}
# PowerShell Example:
Invoke-WebRequest https://api.tumblr.com/v2/blog/rendered-obsolete/posts?api_key=xxx
```

But most of the interesting ones, like [creating a new post](https://www.tumblr.com/docs/en/api/v2#posts---createreblog-a-post-neue-post-format), require OAuth.

## OAuth

Tumblr's [docs](https://www.tumblr.com/docs/en/api/v2#authentication) are a bit light on OAuth 1.0a details, but their implementation closely resembles [Twitter's](https://developer.twitter.com/en/docs/authentication/oauth-1-0a/obtaining-user-access-tokens) and there's also an [RFC](https://datatracker.ietf.org/doc/html/rfc5849).  Twitter calls it "3-legged" OAuth authentication, the RFC calls it "redirection-based" authentication.  Whatever you call it, the gist is:

1. `POST https://www.tumblr.com/oauth/request_token` with client credentials (i.e. consumer key/secret) -> receive "temporary credentials"
1. Direct user to https://www.tumblr.com/oauth/authorize website for "resource owner" approval
1. Once they approve access Tumblr will redirect them to our callback URL -> callback receives "verifier"
1. `GET https://www.tumblr.com/oauth/access_token` with temporary credentials -> receive user credentials (i.e. token and secret)

First, I'll go over the CLI client changes, then we'll deal with the callback since it's an excuse to write an [AWS Lambda in Rust again](https://rendered-obsolete.github.io/2019/03/19/rust_lambda.html).  The final resulting user token/secret can be re-used, so this only needs to be done once per account, thankfully.

## Client

I initially implemented the gory details of OAuth 1.0a using [percent-encoding](https://crates.io/crates/percent-encoding) for string encoding along with [hmac](https://crates.io/crates/hmac) and [sha-1](https://crates.io/crates/sha-1) for the required signing.  But in the end settled on [oauth1-request](https://crates.io/crates/oauth1-request) since it made the client much simpler.

First, use client/consumer credentials to obtain temporary credentials:
```rust
let uri = format!("{}/oauth/request_token", WWW);
let client_credentials =
	oauth1_request::Credentials::new(&self.consumer_key, &self.consumer_secret);
// Sign request using only client/consumer credentials
let auth_header =
	oauth1_request::Builder::<_, _>::new(client_credentials, oauth1_request::HmacSha1)
		// Can optionally specify callback URL here, otherwise Tumblr will use the application default
		//.callback("https://callback_url")
		.post(uri.clone(), &());
let resp = self
	.client
	.post(uri)
	.header(reqwest::header::AUTHORIZATION, auth_header)
	.send()
	.await?;

let resp_body = resp.text().await?;
// Parse `key0=value0&key1=value1&...` in response body for temporary credentials
let mut resp_body_pairs = resp_body
	.split('&')
	.map(|pair| pair.split_once('='))
	.flatten();
let temp_token = get_value(&mut resp_body_pairs, "oauth_token")?.to_owned();
let temp_token_secret = get_value(&mut resp_body_pairs, "oauth_token_secret")?.to_owned();
```

Now we have the "temporary credentials".  We still need the `oauth_verifier` that will be sent to our callback URL.  To receive this from our callback we'll use [AWS Simple Queue Service (SQS)](https://aws.amazon.com/sqs/).

To interact with AWS in Rust there's the [recently announced](https://aws.amazon.com/blogs/developer/a-new-aws-sdk-for-rust-alpha-launch/) official [AWS SDK for Rust](https://github.com/awslabs/aws-sdk-rust) which requires AWS access credentials.  To obtain AWS access credentials, in the AWS console click __$YOUR_NAME (upper-right) > My Security Credentials > Access keys > Create New Access Key__ (keep these safe).  And then set the following [environment variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html):
```powershell
$Env:AWS_ACCESS_KEY_ID="xxx"
$Env:AWS_SECRET_ACCESS_KEY="yyy"
$Env:AWS_DEFAULT_REGION="us-east-1"
```

Now, continuing with the client:
```rust
// Create temporary SQS queue `bullhorn-{temporary token}` to receive oauth_verifier from lambda
let queue_name = format!("bullhorn-{}", temp_token);
let client = aws_sqs::Client::from_env();
let output = client.create_queue().queue_name(queue_name).send().await?;
let queue_url = output.queue_url.unwrap();

// Show "resource owner" approval website in system default web browser
let query = format!("{}/oauth/authorize?oauth_token={}", WWW, temp_token);
let exit_status = open::that(query)?;
```

The ["open" crate](https://crates.io/crates/open) makes it easy to show the Tumblr resource owner approval page in a browser.  While the user is approving our Tumblr application, we wait for our callback to send the oauth verifier via SQS:

```rust
// Receive oauth_verifier from lambda via SQS
let messages = loop {
	let output = client
		.receive_message()
		.queue_url(&queue_url)
		.send()
		.await?;
	if let Some(msgs) = output.messages {
		if !msgs.is_empty() {
			break msgs;
		}
	}
};
// Delete the temporary SQS queue
let _ = client.delete_queue().queue_url(queue_url).send().await?;
let verifier = messages[0].body.as_ref().unwrap();

// Exchange client/consumer and temporary credentials for user credentials
let uri = format!("{}/oauth/access_token", WWW);
let temp_credentials = oauth1_request::Credentials::new(&temp_token, &temp_token_secret);
// Must authenticate with both client and temporary credentials
let token = oauth1_request::Token::new(client_credentials, temp_credentials);
let auth_header =
	oauth1_request::Builder::<_, _>::with_token(token, oauth1_request::HmacSha1)
		// Must include `oauth_verifier`
		.verifier(verifier.as_ref())
		.get(uri.clone(), &());
let resp = self.client
	.get(uri)
	.header(reqwest::header::AUTHORIZATION, auth_header)
	.send()
	.await?;
```

The user token and secret we receive is valid until revoked (via user's __Settings > Apps__), so we can store them somewhere safe to avoid having to do this again.  We can now use Tumblr on the user's behalf.

## Callback

Above covered the important parts of the client.  The part of OAuth authentication remaining is our callback that Tumblr's resource owner approval webpage redirects to.  We could leave an HTTP server running all the time, but that's a bit silly for something we only need occasionally.  This looks like a job for "serverless" and AWS Lambda.

### Basic Lambda

[As before](https://rendered-obsolete.github.io/2019/03/19/rust_lambda.html), we'll use the [AWS Lambda Rust runtime](https://github.com/awslabs/aws-lambda-rust-runtime) to implement our lambda function.  [MUSL](https://musl.libc.org/) is used to create a [fully static Rust binary](https://doc.rust-lang.org/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html).  

In `Cargo.toml`:
```toml
# Rename binary for AWS Lambda custom runtime:
# https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html
[[bin]]
name = "bootstrap"
path = "src/main.rs"

[dependencies]
anyhow = "1.0"
aws_sqs = { git = "https://github.com/awslabs/aws-sdk-rust", tag = "v0.0.10-alpha", package = "aws-sdk-sqs" }
futures = "0.3"
lambda_runtime = "0.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.5.0", features = ["full"] }
tracing = "0.1"
tracing-subscriber = "0.2"
```

In `.cargo/config`:
```toml
[build]
# Override default build target
target = "x86_64-unknown-linux-musl"

# Needed on Mac otherwise build fails with `ld: unknown option: --eh-frame-hdr`
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

For starters, put something like the [basic example](https://github.com/awslabs/aws-lambda-rust-runtime/blob/master/lambda-runtime/examples/basic.rs) in `src/main.rs`:
```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Request {
	oauth_token: String,
	oauth_verifier: String,
}

#[derive(Serialize)]
struct Response {
	msg: String,
}

#[tokio::main]
async fn main() -> Result<(), lambda_runtime::Error> {
	let func = lambda_runtime::handler_fn(handler);
	lambda_runtime::run(func).await
}

async fn handler(event: Request, ctx: lambda_runtime::Context) -> anyhow::Result<Response> {
	Ok(Response{ msg: "ok".to_owned() })
}
```


In order to compile this lambda, we also need to build OpenSSL with musl (based on [this post](https://qiita.com/liubin/items/6c94f0b61f746c08b74c)):
```bash
### Install pre-requisites
# For Mac/OSX: install command line tools and cross-compiler
xcode-select --install
brew install FiloSottile/musl-cross/musl-cross

rustup target add x86_64-unknown-linux-musl

### Build OpenSSL with musl
wget https://github.com/openssl/openssl/archive/OpenSSL_1_1_1f.tar.gz
tar xzf OpenSSL_1_1_1f.tar.gz
cd openssl-OpenSSL_1_1_1f
export CROSS_COMPILE=x86_64-linux-musl-
# `-DOPENSSL_NO_SECURE_MEMORY` is to avoid `define OPENSSL_SECURE_MEMORY` which needs `#include <linux/mman.h>` (which OSX doesn't have).
./Configure CFLAGS="-DOPENSSL_NO_SECURE_MEMORY -fpie -pie" no-shared no-async --prefix=output_abs_path/musl --openssldir=output_abs_path/musl/ssl linux-x86_64
make depend
# Use `sysctl -n hw.physicalcpu` or `hw.logicalcpu` with `-j` if so inclined
make
# Install required stuff to `output_abs_path/musl/` (exclude man-pages, etc.)
make install_sw
# Set value from `--prefix` above
export OPENSSL_DIR=output_abs_path/musl

### Build Rust lambda function
cargo build --release
```

We could now create our lambda with the AWS console, but the aws-lambda-rust-runtime docs show how to do it via the [AWS CLI](https://aws.amazon.com/cli/).

```sh
# Set AWS credentials: https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html
aws configure

# Create package with layout required for custom lambda runtime:
# https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html
# -j/--junk-paths avoids directory structure
zip --junk-paths lambda.zip ../target/x86_64-unknown-linux-musl/release/bootstrap

# Get ARN for desired existing role
aws iam list-roles

# Create new lambda function
aws lambda create-function \
	--function-name bullhorn \
	--handler doesnt.matter \
	--zip-file fileb://./lambda.zip \
	--runtime provided \
	--role arn:aws:iam::01234:role/service-role/bullhorn \
	--tracing-config Mode=Active \
	# Needed with CLI V2 
	--cli-binary-format raw-in-base64-out
	--cli-connect-timeout 10000

# Run the lambda
aws lambda invoke \
	--function-name bullhorn \
	--payload '{"oauth_token": "xxx", "oauth_verifier": "yyy"}' \
	# Needed with CLI V2
	--cli-binary-format raw-in-base64-out \
	response.json

# To update lambda binary
aws lambda update-function-code \
	--function-name bullhorn \
	--zip-file fileb://./lambda.zip \
	--cli-connect-timeout 10000
```

- `--cli-connect-timeout` deals with commands failing with `Error: Connection was closed before we received a valid response` (see [issue](https://github.com/aws/aws-cli/issues/3842))
- Add `--region <region>` for a region other than the default set with `aws configure`

### API Gateway

To get Tumblr to call our lambda via the callback URL we use [AWS API Gateway](https://aws.amazon.com/api-gateway/) to trigger our lambda via an HTTP endpoint.

Open the lambda in AWS console then __+ Add trigger > API Gateway > Create an API__, for "API type" __REST API__ ("HTTP" likely also works) and for "Security" __Open__.

From the list of triggers expand __Details__ and note "API endpoint".  This can be set as "Default callback URL" of the Tumblr application as well as the `oauth_callback` parameter in the client.

When `http://tumblr/authorize` redirects to the callback you get `https://your_callback_url?oauth_token=xxx&oauth_verifier=yyy`.  Apparently lambdas don't accept query strings (the `?xxx=yyy` bits), so we need to [move the query into the body](https://aws.amazon.com/premiumsupport/knowledge-center/pass-api-gateway-rest-api-parameters/).

In AWS console open API Gateway and __/bullhorn ANY > Integration Request__, deselect __Use Lambda Proxy integration__ and expand __Mapping Templates__.  Select __When there are no templates defined__, __Add mapping template__ enter `application/json` and click ☑.  Now there's two options:

- From "Generate template" pick __Method request passthrough__.  This will result in the input to your lambda containing the entire request:

	```json
	transformations: {
	"body-json" : {},
	"params" : {
		"path" : {}
		,"querystring" : {
			"oauth_verifier" : "yyy",
			"oauth_token" : "xxx"
		}
		,"header" : {
		}
	},
	"stage-variables" : {
	},
	"context" : {
		...
	```
- Create a simple mapping to move the query parameters into the request body:

	```json
	{
		"oauth_token": "$input.params('oauth_token')",
		"oauth_verifier": "$input.params('oauth_verifier')"
	}
	```

The latter is easier.  See the [docs for template syntax](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html) details.

Don't forget to __Save__!

In API Gateway you can verify everything works, for "Method" __GET__ and "Query Strings" `oauth_token=xxx&oauth_verifier=yyy` then __Test__.  In the log output you'll see: the original query, the mapped endpoint request body, and the output from the lambda.

Once this is working you must select __Actions > Deploy API__, set "Deployment stage" to __default__ and then __Deploy__.  Failure to do this will result in your lambda receiving the query string from the initial API Gateway configuration.

CloudWatch Logs isn't enabled by default for API Gateway.  [To enable logging](https://aws.amazon.com/premiumsupport/knowledge-center/api-gateway-cloudwatch-logs/) you'll need to create a role and enable them in __API Gateway > {select API} > Stages > default > Logs/Tracing__.


### Final Lambda

In order to work with SQS queues our lambda needs some additional permissions:

1. From AWS console open IAM
1. Select the role used by the lambda function and expand the policy
1. __Edit policy > Add additional permissions__
	- "Service" = __SQS__
	- "Actions" = __Read / GetQueueUrl__ and __Write / SendMessage__
	- "Resources" = `arn:aws:sqs:us-east-1:01234:bullhorn-*` (the naming convention used for our SQS queues)

A working version of the lambda is just the send portion of SQS:
```rust
async fn handler(event: Request, _ctx: lambda_runtime::Context) -> anyhow::Result<Response> {
	let client = aws_sqs::Client::from_env();
	// Get SQS queue based on name set by client
	let queue = client
		.get_queue_url()
		.queue_name(format!("bullhorn-{}", event.oauth_token))
		.send()
		.await?;

	// Send oauth_verifier to client so it can retrieve user token/secret
	let res = client
		.send_message()
		.message_body(event.oauth_verifier)
		.set_queue_url(queue.queue_url)
		.send()
		.await?;
	let response = Response {
		message_id: res.message_id,
		sequence_number: res.sequence_number,
	};
	Ok(response)
}
```

Our [client](#client) will receive the verifier and complete OAuth authentication as described earlier.

Since managed services like SQS generally exhibit [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency), I wonder if `get_queue_url()` can temporarily fail and needs to be retried.  It's worked fine the way it is (so far), so maybe that's accounted for when the client creates the queue?

## Posting to Tumblr

Thus far everything was related to OAuth and obtaining user credentials.  Once we have them we can interact with Tumblr.

The Tumblr API has both a "legacy" and "neue" forms.  Legacy supports markdown, but not canonical URLs.  Neue has canonical URLs but not markdown; posts must be composed from [Neue Post Format (NPF) blocks](https://www.tumblr.com/docs/npf).  It might work to build the markdown as HTML and embed in a block, but I'll look into that later.  Until there's a better solution, we'll create a "link" post that's just a hyper-link to the original article.

```rust
// Check if the article already exists
let posts: Posts = self
	.client
	.get(format!("{}/blog/{}/posts", URL, self.blog_id))
	// Only requires api_key authentication, get response in "Neue Post Format"
	.query(&[("api_key", &self.consumer_key), ("npf", &"true".to_owned())])
	.send()
	.await?
	.json()
	.await?;
let existing = posts.response.posts.iter().find_map(|p| {
	p.content
		.iter()
		// Find block that is a "link" and contains canonical URL, and return its ID
		.find(|block| match block {
			ContentBlock::Link { display_url, .. } => {
				display_url == canonical_url
			}
			_ => false,
		})
		.map(|_| p.id_string.clone())
});

// There's a series of structs for serde to parse the JSON response.
// The following are the important ones:

#[derive(Debug, serde::Deserialize)]
struct Post {
    id_string: String,
    content: Vec<ContentBlock>,
	// <SNIP>
}

// Serde will serialize these from JSON like, e.g.:
// {type="link", display_url="xxx", ...}
#[derive(Debug, serde::Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum ContentBlock {
    Link { display_url: String, title: String },
}
```

Create/update an article:
```rust
// Must authenticate using both client/consumer and user tokens/secrets
let token = oauth1_request::Token::from_parts(
	&self.consumer_key,
	&self.consumer_secret,
	&self.token,
	&self.token_secret,
);
let tags = post.front_matter.tags.map(|tags| RequestTags { tags });
let request = LinkRequest {
	// If we found existing article this will be Some and we'll update.  Otherwise this is None and we create.
	id: existing.clone(),
	title: Some(post.front_matter.title),
	url: canonical_url,
	tags,
	..Default::default()
};

// To create an article: POST {blog_id}/post
// To update: POST {blog_id}/post/edit
let uri = format!(
	"{}/blog/{}/post{}",
	URL,
	self.blog_id,
	if existing.is_some() { "/edit" } else { "" }
);
// Sign the request and create `Authorization` HTTP header
let auth_header =
	oauth1_request::post(uri.clone(), &request, &token, oauth1_request::HmacSha1);
// For POST, request body contains `application/x-www-form-urlencoded`
let body = oauth1_request::to_form_urlencoded(&request);

let resp = self
	.client
	.post(uri)
	.header(reqwest::header::AUTHORIZATION, auth_header)
	.header(
		reqwest::header::CONTENT_TYPE,
		"application/x-www-form-urlencoded",
	)
	.body(body)
	.send()
	.await?;

// HTTP request to create/update "link" type post
#[derive(oauth1_request::Request)]
struct LinkRequest {
    /// Must be `Some` when updating an existing article, `None` when creating a new one
    id: Option<String>,
    #[oauth1(rename = "type")]
    r#type: String,
    tags: Option<RequestTags>,
	
	// <SNIP>
}

// Helper to serialize Vec<_>
struct RequestTags {
    tags: Vec<String>,
}

// Need to impl Display so oauth1_request knows how to serialize a Vec<_>
impl std::fmt::Display for RequestTags {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> Result<(), std::fmt::Error> {
        let tag_param = self.tags.join(",");
        write!(f, "{}", tag_param)
    }
}
```

The resulting post looks like this:

![](https://rendered-obsolete.github.io/assets/tumblr_link_post.png)
