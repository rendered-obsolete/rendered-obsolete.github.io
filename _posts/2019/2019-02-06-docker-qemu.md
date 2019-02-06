---
layout: post
title: Migrating to Docker/Qemu
tags:
- devops
- docker
- qemu
---

We're overhauling our CI testing pipeline.  In [a previous post]({% post_url /2019/2019-02-04-vagrant %}) we looked at using Vagrant to manage Windows VMs.

Here we're using [docker](https://docker.io/) and [QEMU](https://www.qemu.org/) to expand testing to ARM64.  Our examples cover [Rust](https://www.rust-lang.org/) and C#/.NET, but it's easily adaptable to any language/framework of your liking.

## QEMU

We've been wanting to extend testing to more "exotic" platforms, particularly ARM64/aarch64.  [This juicy Travis-CI issue](https://github.com/travis-ci/travis-ci/issues/3376) got us heading in that direction.  They're originally using Debian "Jessie", but "Stretch" is [the first with ARM64 support](https://wiki.debian.org/LTS).

On an Ubuntu 18.04 host, first [install docker on Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/) then:
```
docker run --rm --privileged multiarch/qemu-user-static:register
docker run -it --rm multiarch/debian-debootstrap:arm64-stretch
```

In the resulting shell run `uname -a`:
```
root@f190ea8ef8cc:/# uname -a
Linux f190ea8ef8cc 4.15.0-43-generic #46-Ubuntu SMP Thu Dec 6 14:45:28 UTC 2018 aarch64 GNU/Linux
```

Note __aarch64__.  Pretty slick.

The "magic" here is [Linux binfmt](https://en.wikipedia.org/wiki/Binfmt_misc) which forwards arbitrary executable formats to a user-space application- in this case QEMU.

Some more reading:
- https://www.tomaz.me/2013/12/02/running-travis-ci-tests-on-arm.html
- https://blog.hypriot.com/post/setup-simple-ci-pipeline-for-arm-images/

### Rust

Our first guinea pig is [one of our Rust libraries](https://github.com/jeikabu/runng).  [Here's a Dockerfile](https://github.com/jeikabu/runng/blob/docker_arm64/Dockerfile) to install Rust in the ARM64 environment:
```docker
FROM multiarch/debian-debootstrap:arm64-stretch

RUN apt-get update && apt-get install -y \
    build-essential \
    ca-certificates \
    clang \
    cmake \
    curl

ARG RUST_VER=1.32.0

# Make sure rustup and cargo are in PATH
ENV PATH "~/.cargo/bin:$PATH"
# Install rustup, skip latest toolchain and get a specific version
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain none && \
    ~/.cargo/bin/rustup default $RUST_VER
```

- `clang` and `cmake` are pre-requisites for [runng](https://github.com/jeikabu/runng)
- `ca-certificates` is to deal with curl failing with:
    ```
    ERROR: The certificate of `XXX' is not trusted
    ```

This has been pushed to Docker Hub as [`jeikabu/debian-rust`](https://cloud.docker.com/u/jeikabu/repository/docker/jeikabu/debian-rust) so you can try it with:
```
docker run -it --rm jeikabu/debian-rust:arm64v8-stretch-1.32.0
```

Using `cargo` to run our tests:
```bash
docker run -t -v $(pwd):/usr/src jeikabu/debian-rust:arm64v8-stretch-1.32.0 /bin/bash -c "cd /usr/src; cargo test"
```

NB: The exit code is `0` if all tests succeed.

### .NET Core

Microsoft has an [epic number of images related to .NET Core](https://hub.docker.com/r/microsoft/dotnet).  Of particular interest is the [Dockerfile for .NET Core 3.0 preview targetting ARM64](https://github.com/dotnet/dotnet-docker/blob/master/3.0/sdk/stretch/arm64v8/Dockerfile).  Using that to again enable testing on ARM64 via qemu:

```docker
FROM multiarch/debian-debootstrap:arm64-stretch

RUN apt-get update && apt-get install -y \
    curl \
    gnupg \
    icu-devtools

# From:
# https://github.com/dotnet/dotnet-docker/blob/master/3.0/sdk/stretch/arm64v8/Dockerfile
ENV DOTNET_SDK_VERSION 3.0.100-preview-010184
RUN curl -SL --output dotnet.tar.gz https://dotnetcli.blob.core.windows.net/dotnet/Sdk/$DOTNET_SDK_VERSION/dotnet-sdk-$DOTNET_SDK_VERSION-linux-arm64.tar.gz \
    && dotnet_sha512='3fd7338fdbcc194cdc4a7472a0639189830aba4f653726094a85469b383bd3dc005e3dad4427fee398f76b40b415cbd21b462bd68af21169b283f44325598305' \
    && echo "$dotnet_sha512 dotnet.tar.gz" | sha512sum -c - \
    && mkdir -p /usr/share/dotnet \
    && tar -zxf dotnet.tar.gz -C /usr/share/dotnet \
    && rm dotnet.tar.gz \
    && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet
```

If you're outside North America you might want to use the download URLs in the [release notes](https://github.com/dotnet/core/tree/master/release-notes).  In China, downloading from [https://download.visualstudio.microsoft.com](https://download.visualstudio.microsoft.com) was many times faster than using https://dotnetcli.blob.core.windows.net (like in the official Dockerfile).  YMMV.

This image has also been pushed to Docker Hub as [`jeikabu/debian-dotnet-sdk`](https://cloud.docker.com/u/jeikabu/repository/docker/jeikabu/debian-dotnet-sdk).

Let's run our tests:
```bash
docker run -t -v $(pwd):/usr/src jeikabu/debian-dotnet-sdk:arm64v8-stretch /bin/bash -c "cd /usr/src; dotnet test"
```

`icu-devtools` package is there otherwise you'll get:
```
root@79106a1b502f:/# cd /usr/src
root@864d67ab40bb:/usr/src# dotnet clean
qemu: Unsupported syscall: 283
FailFast:
Couldn't find a valid ICU package installed on the system. Set the configuration flag System.Globalization.Invariant to true if you want to run with no globalization support.

   at System.Environment.FailFast(System.String)
   at System.Globalization.GlobalizationMode.GetGlobalizationInvariantMode()
   at System.Globalization.GlobalizationMode..cctor()
   at System.Globalization.CultureData.CreateCultureWithInvariantData()
   at System.Globalization.CultureData.get_Invariant()
   at System.Globalization.CultureData.GetCultureData(System.String, Boolean)
   at System.Globalization.CultureInfo..ctor(System.String, Boolean)
   at System.Reflection.RuntimeAssembly.GetLocale()
   at System.Reflection.RuntimeAssembly.GetName(Boolean)
   at System.Reflection.Assembly.GetName()
   at System.Diagnostics.Tracing.EventPipeController.GetAppName()
   at System.Diagnostics.Tracing.EventPipeController..ctor()
   at System.Diagnostics.Tracing.EventPipeController.Initialize()
   at System.StartupHookProvider.ProcessStartupHooks()
qemu: uncaught target signal 6 (Aborted) - core dumped
Aborted (core dumped)
```

There's several blogs and StackOverflow questions containing solutions that seem to be variants from the [dotnet documentation for RHEL](https://github.com/dotnet/core/blob/master/Documentation/build-and-install-rhel6-prerequisites.md).  We're not trying to create pedantically diminutive images, so adding `icu-devtools` package will suffice.

## Travis

This should work on _most_ Linux systems.  Let's try running it as part of our [Travis CI](https://travis-ci.org/).

[Create `qemu_arm64.sh`](https://github.com/subor/nng.NETCore/blob/master/scripts/qemu_arm64.sh) to run tests:
```bash
#!/usr/bin/env bash

if [[ "$TRAVIS_OS_NAME" == "linux" ]] || [[ "$OSTYPE" == "linux-gnu" ]]; then
    docker run --rm --privileged multiarch/qemu-user-static:register --reset
    docker run -t -v $(pwd):/usr/src jeikabu/debian-dotnet-sdk:arm64v8-stretch /bin/bash -c "cd /usr/src && dotnet build && dotnet test --filter 'platform!=windows' --verbosity normal"
fi
```

[In `.travis.yml`](https://github.com/subor/nng.NETCore/blob/master/.travis.yml):
```yml
after_success:
  - ./scripts/qemu_arm64.sh
```

NB: if any test fails a non-`0` value will be returned, but [`after_success` won't fail the build](https://docs.travis-ci.com/user/job-lifecycle/#breaking-the-build).  If we decide to keep this and once ARM64 stabilizes we could move it to an earlier phase and let it fail builds.

This is a pretty minimal example using a small project, but still takes just over 5 minutes on Travis.  Larger projects may run afoul of [build timeouts](https://docs.travis-ci.com/user/customizing-the-build/#build-timeouts).  Once again, YMMV.