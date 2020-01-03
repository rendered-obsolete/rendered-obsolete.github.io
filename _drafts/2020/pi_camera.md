---
layout: post
title: Raspberry Pi Zero Camera
tags:
- raspi
- rust
- iot
---

I've got a few people on my team that I've long suspected don't make good use of their time.  Many (most?) days they produce nary a single line of code.

- [Basic camera setup](https://www.raspberrypi.org/documentation/configuration/camera.md)
- [Details on camera utilities](https://www.raspberrypi.org/documentation/raspbian/applications/camera.md)

From looking at the [Pi specifications](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications)
The Zero has nearly the same GPU and video hardware capabilities as the _much_ larger Pi 3:  

| Device | GPU (GFLOPS) | 1080p H.264 Encode (Hz) |
|-|-|-
| RPi 3B | 24 | 60
| Zero W | 28.8 | 30

## Hardware

[Camera Module](https://www.raspberrypi.org/products/camera-module-v2/) or [Pi NoIR Camera](https://www.raspberrypi.org/products/pi-noir-camera-v2/)

1. __Gently__ pull the black plastic tabs to open the connectors on the Pi and camera.  They don't move that far and don't detach:  
![](/assets/pi0_csi_open.jpg)

1. The narrow end of the ribbon (left side of image) goes in the Zero (gold/metaliic contacts down- towards the PCB), the wide end (right side of image) goes in the camera (again, contacts down):  
![](/assets/pi0_camera_ribbon.jpg)

1. At this point you might want to skip ahead to [software](#software) configuration to make sure things are working correctly before going through the trouble of closing the case.

1. If you have [the official case](https://www.raspberrypi.org/products/raspberry-pi-zero-case/), the Zero's micro USB connectors align with the corresponding holes of the bottome of the case:    
![](/assets/pi0_camera_connected.jpg)
The camera lens fits through the hole in the top  
![](/assets/pi0_camera_lens.jpg)

1. That white plastic "wheel" that comes with the camera is used to adjust the [camera focus](https://www.jeffgeerling.com/blog/2017/fixing-blurry-focus-on-some-raspberry-pi-camera-v2-models).  It has little tabs that fit snugly into the notches around the lens:  
![](/assets/pi0_camera_focus.jpg)

We bought two cameras at the same time from the same vendor.  One was so badly out of focus that it was unusable until adjusted.  The second was perfect.

Turn it a few degrees and take screenshots or use the "preview" window to get it right.

## Software

To enable the camera:  
```bash
# Ssh in (replace `raspberrypi.local` with other name or IP)
ssh -X pi@raspberrypi.local

pi$ sudo apt-get update
pi$ sudo apt-get upgrade
pi$ sudo raspi-config
```

In raspi-config: __Interfacing Options > Camera > Yes__

It will insert the following at the bottom of `/boot/config.txt`:
```ini
start_x=1
gpu_mem=128
```

Restart the Zero and verify everything works:
```bash
pi$ sudo shutdown -r now

# Wait... then ssh back in

# Ensure camera was detected
pi$ vcgencmd get_camera
supported=1 detected=1

# Take a picture
pi$ raspistill -o test.jpg
# Open the picture (if you installed "Raspbian Stretch with desktop")
pi$ gpicview test.jpg

# Record 5 second video
pi$ raspivid -t 5000 -o video.h264
```

## MMAL

When you see hardware mention that it supports some video/audio codec(s), it's usually dedicated silicon (often associated with the GPU) that can provide the functionality at minimal CPU/GPU cost other than memory bandwidth.  This is often exposed by a driver or a support library.  For example, AMD Radeon GPUs have the [AMD Display Library (ADL)](https://github.com/GPUOpen-LibrariesAndSDKs/display-library).



[we used ADL]({% post_url /2018/2018-09-09-native-assembly %}#windows-hack)

The Multimedia Abstration Layer (MMAL) is a library providing access to Broadcom GPU features in the Raspberry Pi.  It's a simplified wrapper around [OpenMAX](https://www.khronos.org/openmax/), which is like OpenGL/DirectX for hardware accelerated multimedia (audio/video codecs, processing, etc.).


- [Source for Raspberry Pi GPU libraries](https://github.com/raspberrypi/userland) (including MMAL)
- [mmal-sys](https://github.com/pedrosland/mmal-sys); bindgen generated Rust wrapper to mmal
- [rascam](https://github.com/pedrosland/rascam); high-level wrapper over mmal-sys to interact with camera

