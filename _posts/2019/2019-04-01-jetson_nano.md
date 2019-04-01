---
layout: post
title: Nvidia Jetson Nano
tags:
- nvidia
- linux
- jetson
- sbc
---

![](/assets/nvidia_nano.jpg)

The recent release of the [Jetson Nano](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-nano/) is an inexpensive alternative to Jetson TX1:

Platform | CPU | GPU | Memory | Storage | MSRP
|-|-|-|-|-|-
| Jetson TX1 ([Tegra X1](https://en.wikipedia.org/wiki/Tegra#Comparison)) | 4x ARM Cortex A57 @ __1.73 GHz__ | __256x__ Maxwell @ __998 MHz__ | 4GB LPDDR4 (25.6 GB/s) | __16 GB eMMC__ | $499
| Jetson Nano | 4x ARM Cortex A57 @ 1.43 GHz | 128x Maxwell @ 921 MHz | 4GB LPDDR4 (25.6 GB/s) | Micro SD | __$99__

Basically, for 1/5 the price you get 1/2 the GPU.  [Detailed comparison](https://elinux.org/Jetson) of the entire Jetson line.

The X1 being the SoC that debuted in 2015 with the Nvidia Shield TV:

![](/assets/nvidia_x1.jpg)

__Fun Fact__: During the GDC annoucement when Jensen and Cevat "play" Crysis 3 together their gamepads aren't connected to anything.  Seth W. from Nvidia (Jensen) and I (Cevat) are playing backstage.  _"Pay no attention to that man behind the curtain!"_

## Mini-Rant: Memory Bandwidth

The memory bandwidth of 25.6 GB/s is a little disappointing.  We did some work with K1 and X1 hardware and memory ended up the bottleneck.  It's "conveniently" left out of the above table, but [Xbox 360 eDRAM/eDRAM-to-main/main memory bandwidth is 256/32/22.4 GB/s](https://en.wikipedia.org/wiki/Xbox_360_technical_specifications#Memory_and_system_bandwidth).

Put another way, the TX1's GPU hits 1 TFLOP while the [original Xbox One GPU is 1.31 TFLOPS with main memory bandwidth of 68.3 GB/s](https://en.wikipedia.org/wiki/Xbox_One#Internals) (also ESRAM with over 100 GB/s, _fugged-about-it_).  So, Xbone is 30% higher performance but has almost __2.7x__ the memory bandwidth.

When I heard the Nintendo Switch was using a "customized" X1, I assumed the customization involved a new memory solution.  Nope.  Same LPDDR4 that (imho) would be a better fit for a GPU with 1/4-1/2 the performance.  We haven't done any Switch development, but I wouldn't be surprised if many titles are bottle-necked on memory.  The next most likely culprit being the CPU if overly-dependent on 1-2 threads- but never the GPU.

Looks like we have to hold out until the TX2 to get "big boy pants".  It's 1.3 TFLOPS with 58.3 GB/s of bandwidth (almost 2.3x the X1).

## Setup

Follow the [official directions](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit#write).  On Mac:

```bash
# For the SD card in /dev/disk2
sudo diskutil partitionDisk /dev/disk2 1 GPT "Free Space" "%noformat%" 100%
unzip -p ~/Downloads/jetson-nano-dev-kit-sd-card-image | sudo dd of=/dev/rdisk2 bs=1m
# Wait 10-20 minutes
```

The OS install itself is over 9 GB and the Nvidia demos are quite large such that a 16 GB SD card fills up quick.  We recommend at least 32 GB SD card.

Boots into Nvidia customized Ubuntu 18.04.

Install other software to taste, remember to grab the __arm64__/__aarch64__ version of binaries instead of arm/arm32.

## Benchmarks

For our [Raven Ridge-like](https://en.wikipedia.org/wiki/Ryzen#Raven_Ridge) APU with [Vega GPU](https://en.wikipedia.org/wiki/Graphics_Core_Next#Vega) we run a series of benchmarks:
- 3DMARK, PCMARK, Cinebench
- [Unigine](https://unigine.com/) (Heaven, Valley)
- Several games that have benchmark/demo modes (e.g. ["Rise of the Tomb Raider", "Shadow of Mordor"](https://github.com/subor/sdk/blob/master/docs/topics/optimization.md), etc.)
- Claymore/Phoenix Miner
- A few others

But they're either limited to Windows and/or x86.  Seems the de-facto standard for ARM platforms might be the [Phoronix Test Suite](https://www.phoronix-test-suite.com/).  Fall 2018, [Phoronix did a comparison of a bunch of single-board computers](https://www.phoronix.com/scan.php?page=article&item=16-armlinux-sep2018&num=1) that's not exactly surprising but still interesting.

Purely for amusement also throwing in the results for the [Raspberry Pi Zero]({% post_url /2019/2019-03-28-raspi_zero %}).  Which, to be fair, is in a completely different device class and target use-case.

|Test | Pi Zero | Pi 3 B | Nano | Notes
|-|-|-|-|-|-
| glxgears | 107 | | 2350 | FPS.  When not using [GL/KMS](#glxgears) the Zero only manages 7.7 FPS
| glmark2 | 399 | 383 | 1996 | On the Pi's several tests failed to run
| build runng | 66:00 | 2:15 | 1:08 | Minutes:seconds.  `cargo clean; time cargo build` [runng](https://github.com/jeikabu/runng) [like we did on the Pi Zero]({% post_url /2019/2019-03-28-raspi_zero %}#cross-compiling)

### Phoronix Test Suite

PTS is pretty nice.  It provides an easy way to (re-)run a set of benchmarks based on a unique identifier.  For example, to run the tests from the Fall 2018 ARM article:
```bash
sudo apt-get install -y php-cli php-xml
# Download PTS somewhere and run/compare against article
phoronix-test-suite benchmark 1809111-RA-ARMLINUX005
# Wait a few hours...
# Results are placed in ~/.phoronix-test-suite/test-results/
```

|Test | Pi Zero | Pi 3 B | Nano | TX1 | Notes
|-|-|-|-|-|-
| Tinymembench (memcpy) | 291 | 1297 | 3504 | 3862
| TTSIOD 3D Renderer | | 15.66 | 40.83 | 45.05
| 7-Zip Compression | 205 | 1863 | 3996 | 4526
| C-Ray | | 2357 | 943 | 851 | Seconds (lower is better)
| Primesieve | | 1543 | 466 | 401 | Seconds (lower is better)
| AOBench | | 333 | 190 | 165 | Seconds (lower is better)
| FLAC Audio Encoding | | 387.09 | 103.57 | 78.86 | Seconds (lower is better)
| LAME MP3 Encoding | | 352.66 | 143.82 | 113.14 | Seconds (lower is better)
| Perl (Pod2html) | | 1.2945 | 0.7154 | 0.6007 | Seconds (lower is better)
| PostgreSQL (Read Only) | | 6640 | 12410 | 16079
| Redis (GET) | | 213067 | 568431 | 484688
| PyBench | | 24349 | 7030 | 6348 | ms (lower is better)
| Scikit-Learn | | 844 | 496 | 434 | Seconds (lower is better)

The "Pi 3 B" and "TX1" columns are reproduced from the OpenBenchmarking.org results.  There's also an [older set of benchmarks](https://www.phoronix.com/vr.php?view=24384), `1703199-RI-ARMYARM4104`.

Check out the graphs (_woo hoo!_):
![](/assets/nano_pts.png)

These all seem to be predominantly CPU benchmarks where the TX1 predictably bests the Nano by 10-20% oweing to its 20% higher CPU clock.

Don't let the name "TTSIOD 3D Renderer" fool you, it's a [software renderer](https://github.com/ttsiodras/renderer) (i.e. non-hardware-accelerated; no GPUs were harmed by that test).  Further evidenced by the "Socionext Developerbox" showing.  Socionext isn't some new, up-and-coming GPU company, that device has a __24 core__ ARM Cortex A53 @ 1 GHz (yes, _24_- that's not a typo).

There's [more results for the Nano](https://openbenchmarking.org/result/1903260-HV-JETSONNAN55) including things like Nvidia TensorRT and temperature monitoring both with and without a fan.  But, GLmark2 is likely one of the only things that will run everywhere.

### glxgears

On Nano:
```bash
# Need to disable vsync for Nvidia hardware
__GL_SYNC_TO_VBLANK=0 glxgears
```

On Pi:
```bash
sudo raspi-config
```

__Advanced Options > GL Driver > GL (Full KMS) > Ok__ adds `dtoverlay=vc4-kms-v3d` to the bottom of `/boot/config.txt`.  Reboot and run `glxgears`.

### GLmark2

Getting GLmark2 working on the Nano is easy:
```bash
sudo apt-get install -y glmark2
```

On Pi, [it's currently broken](https://github.com/glmark2/glmark2/issues/80).

You can use the commit right after [Pi3 support](https://github.com/glmark2/glmark2/pull/23/commits/97451860dbee23e925cda8924ee576805603bd8a) was merged:
```bash
sudo apt-get install -y libpng-dev libjpeg-dev
git clone https://github.com/glmark2/glmark2.git
cd glmark2
git checkout 55150cfd2903f9435648a16e6da9427d99c059b4
```

There's a build error:
```
../src/gl-state-egl.cpp: In member function ‘bool GLStateEGL::gotValidDisplay()’:
../src/gl-state-egl.cpp:448:17: error: ‘GLMARK2_NATIVE_EGL_DISPLAY_ENUM’ was not declared in this scope
                 GLMARK2_NATIVE_EGL_DISPLAY_ENUM, native_display_, NULL);
                 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

In `src/gl-state-egl.cpp` at line 427 add:
```c
#else
// Platforms not in the above platform enums fall back to eglGetDisplay.
#define GLMARK2_NATIVE_EGL_DISPLAY_ENUM 0
```

Build everything and run it:
```bash
# `dispmanx-glesv2` is for the Pi
./waf configure --with-flavors=dispmanx-glesv2
./waf
sudo ./waf install
glmark2-es2-dispmanx --fullscreen
```

If it fails with `failed to add service: already-in-use?` take a look at:
- [SO](https://stackoverflow.com/questions/40490113/eglfs-on-raspberry2-failed-to-add-service-already-in-use)
- [Forum post](https://www.raspberrypi.org/forums/viewtopic.php?p=1274884)

Both mention commenting out `dtoverlay=vc4-kms-v3d` in `/boot/config.txt`- which was added when we enabled "GL (Full KMS)".

## Hello AI World

After getting your system setup, take a look at ["Hello AI World"](https://github.com/dusty-nv/jetson-inference) which does image recognition and is pre-trained with 1000 objects.  Start with "Building the Repo from Source".  It took a while to install dependencies, but then everything builds pretty quick.

```bash
cd jetson-inference/build/aarch64/bin
# Recognize what's in orange_0.jpg and place results in output.jpg
./imagenet-console orange_0.jpg output.jpg

# If you have a camera attached via CSI (e.g. Raspberry Pi Camera v2)
./imagenet-camera googlenet # or `alexnet`
```