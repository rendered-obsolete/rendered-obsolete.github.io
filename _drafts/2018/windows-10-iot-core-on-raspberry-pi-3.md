---
layout: post
title: Windows 10 IoT Core on Raspberry Pi 3
tags:
- iot
- raspi
- iotcore
- win10
---

Following up on [an earlier post]({% post_url /2017/2017-11-23-win10-iot-core-redux %}), finally got around to trying to install IoT Core on my actual Raspberry Pi device:
- [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
- [7" Touchscreen](https://www.raspberrypi.org/products/raspberry-pi-touch-display/)

My development machine is a [15" Macbook Pro](https://support.apple.com/kb/SP756?locale=en_US) running OSX High Sierra (10.13.6).  Wasn't able to get a Windows 10 VM in VirtualBox to recognize the SD card slot in my adapter (the otherwise fantastic [Satechi type-C multi-port adapater](https://satechi.net/collections/usb-type-c/products/satechi-aluminum-multi-port-adapter-4k)), so started looking for how to flash the image without using [IoT Core Dashboard](https://developer.microsoft.com/en-us/windows/iot/Downloads).

## Image Preparation

Microsoft has some docs about using [dism](https://docs.microsoft.com/en-us/windows/iot-core/connect-your-device/dism) instead of Dashboard.  From the [IoT Core download page](https://developer.microsoft.com/en-us/windows/iot/Downloads) you can obtain an iso file containing an msi that by default installs the image to: `C:\Program Files (x86)\Microsoft IoT\FFU\RaspberryPi2\flash.ffu`.  ffu2img ([https://github.com/t0x0/random/wiki/ffu2img](https://github.com/t0x0/random/wiki/ffu2img)) can be used to convert the [ffu](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-using-full-flash-update--ffu) to an img file usable with [dd](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dd.1.html) found on OSX (and Linux, etc.).

Ensure you're using [Python 2.7](https://www.python.org/download/releases/2.7/) and convert the image:
```
python ffu2img.py <path>/flash.ffu
```

Resulting in a __flash.img__ file in the same location as the ffu.

## Writing Image to SD Card

To write the image to an SD card I consulted the documentation at [raspberrypi.org](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md).

I had trouble getting those instructions to work with my card/reader.  No matter what, dd failed with `Operation not permitted`.  In the end I had to:
1. Insert card into reader and connect reader to laptop
1. Without unmounting the SD card, run: `sudo diskutil partitionDisk /dev/disk2 1 MBR "Free Space" "%noformat%" 100%`
1. Disconnect then reconnect the reader (above command seemed to eject the SD card)
1. Write the image: `sudo dd bs=1m if=PATH/flash.img of=/dev/rdisk2 conv=sync`

In my case, the SD card is __/dev/disk2__.  /dev/disk2 and /dev/rdisk2 in the above commands should be adjusted appropriately for other cases.

Using df can see several partitions were created (/dev/disk2s1, disk2s2, disk2s3, and disk2s6) on the SD card.  Once inserted in the Raspberry Pi device and powered on, IoT Core should boot.

## IoT Core Setup

As it happens, I didn't have a USB keyboard on hand to get the device on the wifi.

Using an ethernet cable to connect the device directly to my laptop, I obtained the device's link-local address from the bottom-left of the touch screen.  The device's admin portal is now accessible by pointing my laptop browser at `http://169.254.236.118:8080` (default username/password is Administrator/p@ssw0rd).

In the left panel, selecting Connectivity, then Network, you can configure the wifi AP:  
![]({{ "/assets/iot_dashboard_network.png" | absolute_url }})

Initially the display was upside-down (assuming the HDMI port is "up" as when using [this touchscreen case](https://www.amazon.com/Raspberry-Pi-7-Inch-Touch-Screen/dp/B01GQFUWIC)).  To fix this, in the left panel pick __Device Settings__ and towards the bottom in __Display Orientation__ select `Landscape (Flipped)`:  
![]({{ "/assets/iot_dashboard_orientation.png" | absolute_url }})