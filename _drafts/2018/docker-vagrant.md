---
layout: post
title: Migrating to Vagrant/Docker/QEMU
tags:
- devops
- docker
- vagrant
- qemu
---

Need testing various version of Windows: 7, 10, 10 LTSB ("IoT Enterprise").

Previously Jenkins [node/agent label](https://jenkins.io/doc/book/pipeline/syntax/#agent) physical machines that were manually configured:
- Scalability: static set of machines that became a bottleneck before releases
- Fail-fast/Debuggable: had to wait for PR to percolate through build pipeline to find out it didn't work and then take the node offline to debug
- Reproducable: manually configured error-prone, people would make changes to the environment, etc.

[AppVeyor]({% post_url /2018/2018-09-21-dotnet %})
[Travis]({% post_url /2018/2018-09-30-rust-ffi-ci %})


## Windows VM

Vagrant Windows guest VMs
https://www.vagrantup.com/docs/boxes/base.html#windows-boxes

We're in the process of evaluating Windows 10 Enterprise LTSC (the OS formerlly known as Windows 10 "IoT Enterprise"/LTSB) which is comparable to Windows 10 Enterprise version 1809/RS5.  I'm installing to VirtualBox as `win10_ltsc_2019`.

### WinRM

[Windows Remote Mangement](https://docs.microsoft.com/en-us/windows/desktop/WinRM/portal)

Make current network connection "Private" otherwise the WinRM configuration will fail: 
```powershell
Get-NetConnectionProfile
# Get "Name" value from output
Set-NetConnectionProfile -Name "Name from above" -NetworkCategory Private
```

`vagrant resume` there's a problem with WinRM:
```
default: WinRM transport: negotiate
```

In my case I'm on a domain and the connection will eventually rename itself.  Once that happens I need to set the category to "Private" again.

### RDP/SSH

[Enable Remote Desktop](https://docs.microsoft.com/en-us/windows-server/remote/remote-desktop-services/clients/remote-desktop-allow-access) for `vagrant rdp` to work.

In order for `vagrant ssh` to work, must install OpenSSH server:
- Windows 10 RS3/1709 and later, it's an [optional component](https://blogs.msdn.microsoft.com/powershell/2017/12/15/using-the-openssh-beta-in-windows-10-fall-creators-update-and-windows-server-1709/)
- For ealier versions including Windows 10 LTSB 2016 (RS1), https://github.com/PowerShell/Win32-OpenSSH/releases

In either case, you also need to:
- [Add the vagrant public key](https://www.vagrantup.com/docs/boxes/base.html#default-user-settings) to `~/.ssh/authorized_keys` (see [this SO](https://stackoverflow.com/questions/16212816/setting-up-openssh-for-windows-using-public-key-authentication))
- Have sshd start automatically: `sc config sshd start= auto`

### Boxing

At this point a PowerShell guru would probably head right into provisioning.  [With AppVeyor]({% post_url /2018/2018-09-21-dotnet %}#code-coverage) I got some mileage out of [Chocolatey](https://chocolatey.org/), but my shell-fu is woefully inadequate to fully automate Windows.

A very common Visual Studio:
- [Full Visual Studio](https://visualstudio.microsoft.com/downloads/)
- ["BuildTools"](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2017) (a streamlined version of VS)
- [Visual Studio container](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container?view=vs-2017)


Working with boxes is straight-forward.  In the case of virtual box:
https://www.vagrantup.com/docs/virtualbox/boxes.html#packaging-the-box
```bash
# Save the base image somewhere
vagrant package --base win10_ltsc_2019 --output /shared_folder/win10_ltsc_2019.box

# Remove the box if it already exists
vagrant box remove win10_ltsc_2019
# Add the saved base image to list of available "boxes"
vagrant box add --name win10_ltsc_2019 /shared_folder/win10_ltsc_2019.box
```

Keep in mind none of this is fast as it involves 5-15GB VM images.  This should be used for relatively "static" configuration; I wouldn't want to mess with it every day or even every week.  For things that change frequently (like your software) there's "shared folders".

Neat scripts and Vagrantfiles in this repo for working with Windows and Docker:
https://github.com/StefanScherer/docker-windows-box


## Vagrantfile

```ruby
Vagrant.configure("2") do |config|
    config.vm.box = "win10_ltsc_2019"
    config.vm.guest = :windows
    config.vm.communicator = "winrm"
    # 3389 RDP
    config.vm.network "forwarded_port", guest: 3389, host: 3389
end
```

`vagrant up`:
```
==> default: Forwarding ports...
    default: 3389 (guest) => 3389 (host) (adapter 1)
    default: 5985 (guest) => 55985 (host) (adapter 1)
    default: 5986 (guest) => 55986 (host) (adapter 1)
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Mounting shared folders...
    default: /vagrant => /Users/jake/projects/nng.NETCore
```

Port `3389` is RDP so `vagrant rdp` works.  I added a (very belated) [answer to Stack Overflow](https://stackoverflow.com/questions/28906432/vagrant-rdp-windows2012r2-how-do-i-rdp-into-my-vagrant-box) about this:

```
$ vagrant rdp
==> default: Detecting RDP info...
    default: Address: 127.0.0.1:3389
    default: Username: vagrant
```

Likewise, `vagrant ssh`:  
![]({{ "/assets/vagrant_ssh_win10.png" | absolute_url }})

## QEMU

Been wanting to extend testing to more "exotic" platforms, particularly ARM64/aarch64.  [This juicy Travis-CI issue](https://github.com/travis-ci/travis-ci/issues/3376) got us heading in that direction.

Originally using Debian "Jessie", but "Stretch" is [first with ARM64 support](https://wiki.debian.org/LTS).

Working on Ubuntu 18.04 host, first [install docker on Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/) then:
```
sudo apt-get install qemu-system-aarch64
docker run --rm --privileged multiarch/qemu-user-static:register
docker run -it --rm multiarch/debian-debootstrap:arm64-stretch
```

In that shell run `uname -a`:
```
root@f190ea8ef8cc:/# uname -a
Linux f190ea8ef8cc 4.15.0-43-generic #46-Ubuntu SMP Thu Dec 6 14:45:28 UTC 2018 aarch64 GNU/Linux
```

Note __aarch64__.  Pretty slick.

### Rust

[Dockerfile](https://github.com/jeikabu/runng/blob/docker_arm64/Dockerfile):
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

This has been pushed to [Docker Hub](https://cloud.docker.com/u/jeikabu/repository/docker/jeikabu/debian-rust) so you can try it via:
```
docker run -it --rm jeikabu/debian-rust:arm64v8-stretch-1.32.0
```

https://www.tomaz.me/2013/12/02/running-travis-ci-tests-on-arm.html
https://blog.hypriot.com/post/setup-simple-ci-pipeline-for-arm-images/
https://hub.docker.com/r/arm64v8/rust/

### .NET Core

[Dockerfile ARM64](https://github.com/dotnet/dotnet-docker/blob/master/3.0/sdk/stretch/arm64v8/Dockerfile)

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

If you're outside North America you might want to use the download URLs in the [release notes](https://github.com/dotnet/core/tree/master/release-notes).  In China, downloading from [https://download.visualstudio.microsoft.com](https://download.visualstudio.microsoft.com) was many times faster than https://dotnetcli.blob.core.windows.net like in their Dockerfile.  YMMV.

Let's run our tests:
```bash
docker run -t -v $(pwd):/usr/src jeikabu/debian-dotnet-sdk:arm64v8-stretch /bin/bash -c "cd /usr/src; dotnet test"
```

One last comment.  `icu-devtools` package is there otherwise you'll get:
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

There's several blogs and StackOverflow questions containing solutions that seem to be variants from the [dotnet documentation for RHEL](https://github.com/dotnet/core/blob/master/Documentation/build-and-install-rhel6-prerequisites.md).  Not trying to create pedantically tiny images, so adding `icu-devtools` package will suffice.

### Travis

