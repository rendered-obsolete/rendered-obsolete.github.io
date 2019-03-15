---
layout: post
title: Raspberry Pi 3/Zero Raspbian Primer
tags:
- raspi
- linux
---

That day has come.

I've had a Raspberry Pi 3 B with a touchscreen [running Windows 10 IoT Core]({% post_url /2018/2018-09-02-win10-iot-core-raspi %}) on my desk at work for a while now.  IoT Core isn't dead, I see it reboot for updates every now and then.

https://github.com/Microsoft?utf8=%E2%9C%93&q=iotcore&type=&language=

[Raspbian](https://www.raspberrypi.org/downloads/).

# Raspberry Pi 3

## SSH and VNC
Pretty much the first thing I ever do is get SSH working so I can use a single keyboard/mouse/screen/etc.


```
sudo raspi-config
```

- _Interfacing Options > VNC > Yes_
- _Interfacing Options > SSH > Yes_

When it comes to Linux configuration, one of the best investments you can make is setting up "SSH Key-Based Authentication".  If that means nothing to you look into it, it will change your life.  There's numerous excellent guides on the internet, but if you're on Linux/macOS you should be able to:
```bash
# Replace with IP address of your Pi
ssh-copy-id pi@192.168.X.Y
```

Speaking of investments, given the premium placed on screen real-estate don't overlook the `-X` option to ssh (on macOS/OSX you also need [XQuartz](https://www.xquartz.org/)):  

![](/assets/ssh_x_pi.png)

xrdp vs VNC

If you're using a VNC client other than RealVNC you may not be able to connect.  TigerVNC complains: `No matching security types`.
https://www.raspberrypi.org/documentation/remote-access/vnc/

## Firmware

```
sudo rpi-update
```

## Screen

If you've got some kind of small display (like the [7" touchscreen](https://www.raspberrypi.org/products/raspberry-pi-touch-display/))

- Rotate screen 180 degrees.  In `/boot/config.txt` add:
    ```
    lcd_rotate=2
    ```
- Adjust screen brightness (max brightness is `255`, higher than that results in "I/O Error"):
    ```
    sudo sh -c "echo 80 > /sys/class/backlight/rpi_backlight/brightness"
    ```
- Virtual Keyboard.  Regardless if you call it a "soft keyboard", "on-screen keyboard", or something else, you might want one.  Regardless if you prefer iOS or Android, prepare to be disappointed:
    ```
    sudo apt-get install matchbox-keyboard
    ```

__General Linux Goodness__

In `~/.inputrc`:
```
set completion-ignore-case on
```

## Software

Pretty much the first thing you're going to want to do is:
```bash
sudo apt-get update
sudo apt-get install -y vim screen
```

- [Visual Studio Code](https://github.com/headmelted/codebuilds/releases)
- [PowerShell Core](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-linux?view=powershell-6#raspbian)


# Zero

Somewhat on impulse I ordered a [Raspberry Pi Zero "W"](https://www.raspberrypi.org/products/raspberry-pi-zero/).  Not sure how I hadn't heard about this until now (first came out in 2015, "Zero W" with Wifi/Bluetooth in 2017), but as soon as I saw it I had to have one:

![](/assets/raspi_zero.jpg)

Way back in 2000 or so I did a project with wireless "Motes" running [TinyOS](https://en.wikipedia.org/wiki/TinyOS):

![](/assets/mote.jpg)

I can't remember which exact model we had but they all seem to have had around 4 __KB__ of SRAM and 512 __KB__ flash memory.  Here we are 20 years later and the Zero is an absolute beefcake with 512 MB RAM and a 16 GB micro SD card (not to mention a single-core 32-bit ARM CPU).

- [Raspberry Pi OTG Mode](https://gist.github.com/gbaman/50b6cca61dd1c3f88f41)
- [Setting Up Pi OTG - The Quick Way](https://gist.github.com/gbaman/975e2db164b3ca2b51ae11e45e8fd40a)
- Adafruit guide https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget/overview

One important difference is the Zero is __armv6__ ([like the Raspberry Pi 1](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications)) instead of __armv7__ like the Raspberry Pi 3.  This means the provided .NET Core, PowerShell, and VS Code won't work.  I may need to look into building .NET Core from source, but I doubt VS Code would run well, TBH.

Thankfully, Rust works!  All we really need is Rust anyway.

4.14.79 -> 4.19.27

Later part of a DARPA project involving sensor network of radar nodes, "ubiquitous computing" laboratory.
