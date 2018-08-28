---
layout: post
title: Windows 10 IoT Core on Raspberry Pi 3 (2018)
tags:
- iot
- raspi
- iotcore
- win10
---

Decided to go back and re-visit my [post on installing Win10 IoT Core on a Raspberry Pi 3]({% post_url /2017/2017-12-25-windows-10-iot-core-on-raspberry-pi-3 %}) using OSX (without using [IoT Core Dashboard](https://developer.microsoft.com/en-us/windows/iot/Downloads)).  Updated to use the latest versions of: OSX, IoT Core, online resources, etc.

Using mostly the same devices as last time:
- [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
- [7" Touchscreen](https://www.raspberrypi.org/products/raspberry-pi-touch-display/)
- [15" Macbook Pro](https://support.apple.com/kb/SP756?locale=en_US) running OSX High Sierra (10.13.6)
- [Satechi Multi-Port Adapater V2](https://www.amazon.com/Satechi-Aluminum-Multi-Port-Ethernet-Pass-Through/dp/B075FW7H5J/ref=sr_1_3?ie=UTF8&qid=1534991689&sr=8-3&keywords=satechi+v2) (USB-C)

## Image Preparation

Microsoft has some docs about using [dism](https://docs.microsoft.com/en-us/windows/iot-core/connect-your-device/dism) instead of Dashboard.

Similar to last time, we need to obtain an IoT Core image (an [ffu](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-using-full-flash-update--ffu))
and then convert it to an img usable with [dd](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dd.1.html) (found on OSX, Linux, etc.).

1. Locate IoT Core Build:
    - [Try this link](https://go.microsoft.com/fwlink/?LinkId=846058)
    - Should that not work, from the [IoT Core download page](https://developer.microsoft.com/en-us/windows/iot/Downloads) under "Latest Windows 10 IoT Core Builds" there should he a "For RaspBerry Pi 2 & 3"
1. Download the iso (at the time of writing `17134.180410-1804.rs4_release_amd64fre_IOTCORE_RPi.iso`) __to a Windows machine__
1. Double-click iso to mount it and run `Windows_10_IoT_Core_for_RPi.msi` inside
1. Click through installer to install files to `C:\Program Files (x86)\Microsoft IoT\FFU\RaspberryPi2\`.
1. Download [ffu2img.py](https://github.com/t0x0/random) ([docs](https://github.com/t0x0/random/wiki/ffu2img)) 
1. Ensure you're using [Python 2.7](https://www.python.org/download/releases/2.7/) and convert the image:
    ```
    python ffu2img.py <path>/flash.ffu [output_filename]
    ```

The image conversion should take a minute or two and result in a __flash.img__ file in the same location as the ffu (if you omit `output_filename`).

The ffu2img repository also contains a py3 script for python 3, but I didn't try it.

## Write Image to SD Card and Boot

This time the [raspberrypi.org Mac instructions](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md) work without issue.

- My SD card is __/dev/disk4__
- I made sure to __unmount__ (not _eject_) all `disk4` volumes (in __Disk Utility__ select a volume and check __Device__.  One or more may say `disk4sX`)
-  Write the img with `sudo dd bs=1m if=PATH/flash.img of=/dev/rdisk4 conv=sync` (takes around 5 minutes)
- In __Disk Utility__ (or by running `df`) verify several partitions were created (/dev/disk4s1, disk4s2, and disk4s5) on the SD card
- Eject all the volumes, insert SD card back in the Raspberry Pi, and power on

## IoT Core Setup

[Last year I wrote]({% post_url /2017/2017-12-25-windows-10-iot-core-on-raspberry-pi-3 %}):
```
As it happens, I didn't have a USB keyboard on hand to get the device on the wifi.

Using an ethernet cable to connect the device directly to my laptop, I obtained the device's link-local address from the bottom-left of the touch screen.  The device's admin portal is now accessible by pointing my laptop browser at `http://169.254.236.118:8080` (default username/password is Administrator/p@ssw0rd).
```

This time I came preparred, but I suspect the same gymnastics would work.

Connect the Pi via ethernet, got a DHCP address (`192.168.2.39`), and pointed a browser at `192.168.2.39:8080` (default username/password is same as before).  The rest is basically the same:

In the left panel, selecting __Connectivity-> Network__, you can configure the wifi AP:  
![]({{ "/assets/iot_dashboard_network.png" | absolute_url }})

Initially the display was upside-down (assuming the HDMI port is "up" as when using [this touchscreen case](https://www.amazon.com/Raspberry-Pi-7-Inch-Touch-Screen/dp/B01GQFUWIC)).  To fix this, in the left panel pick __Device Settings__ and towards the bottom in __Display Orientation__ select `Landscape (Flipped)`:  
![]({{ "/assets/iot_dashboard_orientation.png" | absolute_url }})

Now to finally get started on that long over-due project.