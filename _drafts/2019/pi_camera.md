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
The Zero has nearly the same GPU and video hardware capabilities as the larger Pi 3:
- GPU: 24 vs 28.8 GFLOPS
- Encode/decode: 1080p 30 Hz vs 60 Hz H.264

## Hardware

- [Camera Module](https://www.raspberrypi.org/products/camera-module-v2/) or [Pi NoIR Camera](https://www.raspberrypi.org/products/pi-noir-camera-v2/)

1. Gently pull the dark plastic to open the connector.  It doesn't move that far and it doesn't detach.
![](/assets/pi0_csi_open.jpg)

1. The narrow end (left) goes in the Zero (gold/metaliic contacts down- towards the PCB), the wide end (right) goes in the camera (again, contacts down):  
![](/assets/pi0_camera_ribbon.jpg)

1. At this point you might want to skip ahead to [software](#software) configuration to make sure things are working correctly before going through the trouble of closing the case.

1. If you have a case, the micro USB connectors align with the corresponding holes of the bin  
![](/assets/pi0_camera_connected.jpg)
The camera lens fits through the hole in the top  
![](/assets/pi0_camera_lens.jpg)

1. That white plastic "wheel" that comes with the camera is used to adjust the [camera focus](https://www.jeffgeerling.com/blog/2017/fixing-blurry-focus-on-some-raspberry-pi-camera-v2-models).  It has little notches that fit snuggly into the grooves around the lens:
![](/assets/pi0_camera_focus.jpg)

We bought two cameras at the same time from the same vendor.  One was so badly out of focus that it was unusable until adjusted.  The second was perfect.

Turn it a few degrees and take screenshots or use the "preview" window to get it right.

## Software

```bash
# Ssh in (replace `raspberrypi.local` with other name of IP)
ssh -X pi@raspberrypi.local

pi$ sudo apt-get update
pi$ sudo apt-get upgrade

pi$ sudo raspi-config
```

__Interfacing Options > Camera > Yes__

It will insert the following at the bottom of `/boot/config.txt`:
```
start_x=1
gpu_mem=128
```

Restart the Zero.

Verifying everything works:
```bash
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


- [Source for Raspberry Pi GPU libraries](https://github.com/raspberrypi/userland) (including MMAL)
- [mmal-sys](https://github.com/pedrosland/mmal-sys); bindgen generated Rust wrapper to mmal
- [rascam](https://github.com/pedrosland/rascam); high-level wrapper over mmal-sys to interact with camera