---
layout: post
title: Raspberry Pi Zero Rasbian Primer
tags:
- raspi
- linux
- rust
---

Somewhat on impulse I ordered a [Raspberry Pi Zero "W"](https://www.raspberrypi.org/products/raspberry-pi-zero/).  Not sure how I hadn't heard about this until now (Zero first came out in 2015, "Zero W" with Wifi/Bluetooth in 2017), but as soon as I saw it I had to have one:

![](/assets/raspi_zero.jpg)

Dawww... look at that lil' guy.

Way back in early 2000's I did a project with wireless "Motes" running [TinyOS](https://en.wikipedia.org/wiki/TinyOS):

![](/assets/mote.jpg)

I can't remember which exact model we had but they all seem to have had around 4 __KB__ of SRAM and 512 __KB__ flash memory.  Here we are nigh 20 years later and the Zero is an absolute beefcake with 512 MB RAM, micro SD slot (here 16 GB), and paired with a single-core 32-bit ARM CPU.

## Setup

1. [Download Raspbian](https://www.raspberrypi.org/downloads/)
1. [Install](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)

Prior to the Zero W/WH with wifi, there was an approach to configure the system completely headless and get it on the network that's still pretty useful:
- [Raspberry Pi OTG Mode](https://gist.github.com/gbaman/50b6cca61dd1c3f88f41)
- [Setting Up Pi OTG - The Quick Way](https://gist.github.com/gbaman/975e2db164b3ca2b51ae11e45e8fd40a)
- Adafruit guide https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget/overview

For macOS:
```bash
# Write raspbian image to SD in /dev/disk2
diskutil unmountDisk /dev/disk2
sudo dd bs=1m if=~/Downloads/2018-11-13-raspbian-stretch.img of=/dev/rdisk2 conv=sync
cd /Volumes/boot
# Enable SSH
touch ssh
vim config.txt # Add to the bottom: dtoverlay=dwc2
vim cmdline.txt # After `rootwait` add: modules-load=dwc2,g_ether
sudo diskutil eject /dev/rdisk2
```

Insert the SD card into the Pi Zero and connect your PC to the micro USB port labeled "USB" (it's the one closest to the middle of the unit) to power it on.

After ~30 seconds, in __System Preferences > Network__ you should see:
![](/assets/osx_rndis_gadget.png)

Connect with:
```bash
# Setup ssh key authentication.  Default password is `raspberry`
ssh-copy-id pi@raspberrypi.local
ssh -X pi@raspberrypi.local
```

If you've got multiple devices, or you re-flash at some point you'll see the very scary:
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@       WARNING: POSSIBLE DNS SPOOFING DETECTED!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
The ECDSA host key for raspberrypi.local has changed,
and the key for the corresponding IP address 169.254.28.197
is unknown. This could either mean that
DNS SPOOFING is happening or the IP address for the host
and its host key have changed at the same time.
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.
Please contact your system administrator.
Add correct host key in /Users/XXX/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/XXX/.ssh/known_hosts:37
ECDSA host key for raspberrypi.local has changed and you have requested strict checking.
Host key verification failed.
```

Find the line like `Offending ECDSA key in /Users/XXX/.ssh/known_hosts:37`.  Edit `~/.ssh/known_hosts` and remove line `37` to fix the nasty.

![](/assets/osx_rndis_sharing.png)

To get it on the network, open __System Preferences > Sharing__.
For __Share your connection from__ select your computer's internet connection (probably either LAN or Wifi) and __To computers using__ select the `RNDIS/Ethernet Gadget`.

If the wifi uses 802.1x authentication (like my office) connection sharing is out.

## Software

One important difference is the Zero is __armv6__ (with VFPv2 but no NEON- [like the Raspberry Pi 1](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications)) instead of __armv7__ like the Raspberry Pi 3.  This means the available .NET Core, PowerShell, VS Code and other binaries won't "just" work.  I may look into building 32-bit .NET Core from source, but with a single core puttering along at 1 GHz and only 512 MB of RAM I doubt VS Code would run well, TBH.

Remember when I advocated learning vim or emacs?  Guess what works beautifully on low-end devices.

Thankfully, Rust also works!  All we really need is Rust anyway.

## Cross-Compiling Rust

It's easy to install Rust 1.33 via rustup:
```bash
curl https://sh.rustup.rs -sSf | sh
```

Let's build [runng](https://github.com/jeikabu/runng).

On my 2017 MacBook Pro (dual-core 2.5 GHz Intel Core i7 with 16 GB 2133 MHz LPDDR3), `cargo clean; cargo build` takes about 60 __seconds__.  
On the Pi Zero, it takes 66 __minutes__.  _Ouch_!

This looks like a job for cross-compiling!  There's a bunch of posts about using [crosstool-NG](http://crosstool-ng.github.io/), but we're going to do something a little different.  Afterall, who wants to deal with shared libraries?  Enter [musl](https://www.musl-libc.org/).

Let's start with something simple, a "hello world" binary:

```bash
cargo new --bin hello_muscles
cd hello_muscles/
```

In `hello_muscles/.cargo/config` (or `~/.cargo/config`):
```toml
[target.arm-unknown-linux-musleabihf]
linker = "arm-linux-musleabihf-gcc"
```

```bash
# On OSX: install arm/x86_64 linux-musl cross-compilers
brew install FiloSottile/musl-cross/musl-cross --with-arm-hf
# Install rust toolchain
rustup target add arm-unknown-linux-musleabihf
CROSS_COMPILE=arm-linux-musleabihf- cargo build --release --target arm-unknown-linux-musleabihf
```

Check `target/arm-unknown-linux-musleabihf/release/` for output:
```bash
$ file target/arm-unknown-linux-musleabihf/release/hello_muscles
target/arm-unknown-linux-musleabihf/release/hello_muscles: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, with debug_info, not stripped

$ ls -l target/arm-unknown-linux-musleabihf/release/hello_muscles
-rwxr-xr-x  2 jake  staff  2674552 Mar 18 16:28 target/arm-unknown-linux-musleabihf/release/hello_muscles
```

A ~2.7MB self-contained ARM executable.  Let's give it a whirl on the Zero:
```bash
$ scp target/arm-unknown-linux-musleabihf/release/hello_muscles pi@raspberrypi.local:~/
$ ssh pi@raspberrypi.local

pi@raspberrypi:~ $ ./hello_muscles 
Hello, world!
```

Sweet, sweet nectar!

We went through that pretty quick.  It's no mere coincidence to use musl (although my love/hate relationship with shared libs is well-founded), we're actually already using it elsewhere on another tangent to this project.  I stole the key bits from that as-of-yet-unfinished post, but we'll be coming back to this _very_ soon.