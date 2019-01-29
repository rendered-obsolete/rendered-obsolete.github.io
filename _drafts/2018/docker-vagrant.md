---
layout: post
title: Docker and Vagrant
tags:
- devops
- docker
- vagrant
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

### WinRM

If necessary, make current network connection "Private" otherwise the WinRM configuration will fail: 
```powershell
Get-NetConnectionProfile
# Get "Name" value from output
Set-NetConnectionProfile -Name "Name from above" -NetworkCategory Private
```

### SSH/RDP

In order for `vagrant ssh` to work, must install OpenSSH server:
- Windows 10 RS3/1709 and later, it's an [optional component](https://blogs.msdn.microsoft.com/powershell/2017/12/15/using-the-openssh-beta-in-windows-10-fall-creators-update-and-windows-server-1709/)
- For ealier versions including Windows 10 LTSB 2016 (RS1), https://github.com/PowerShell/Win32-OpenSSH/releases

In either case, [you need to add the vagrant public key](https://www.vagrantup.com/docs/boxes/base.html#default-user-settings) to `~/.ssh/authorized_keys`:
https://stackoverflow.com/questions/16212816/setting-up-openssh-for-windows-using-public-key-authentication

While you're at it enable Remote Desktop.

### Boxing

At this point a PowerShell guru would probably head right into provisioning.  [With AppVeyor]({% post_url /2018/2018-09-21-dotnet %}#code-coverage) I got some mileage out of [Chocolatey](https://chocolatey.org/), but my shell-fu is woefully inadequate to automate Windows machine setup.

Neat scripts and Vagrantfiles in this repo for working with Windows and Docker:
https://github.com/StefanScherer/docker-windows-box

Working with boxes is straight-forward.  In the case of virtual box:
https://www.vagrantup.com/docs/virtualbox/boxes.html#packaging-the-box
```bash
# Save the base image somewhere
vagrant package --base win10_ltsc_2019 --output /shared_folder/win10_ltsc_2019.box
# Add the saved base image to list of available "boxes"
vagrant box add --name win10_ltsc_2019 /shared_folder/win10_ltsc_2019.box
```

Keep in mind none of this is fast as it involves 5-15GB VM images.  This should be used for relatively "static" configuration; I wouldn't want to mess with it every day or even every week.

__Installing Visual Studio__
https://www.microsoft.com/net/download (or [archives](https://www.microsoft.com/net/download/archives))

Visual Studio container
https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container?view=vs-2017


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

Been wanting to extend testing to more "exotic" platforms like ARM64/aarch64.  [This juicy Travis-CI issue](https://github.com/travis-ci/travis-ci/issues/3376) got us heading in that direction.

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
