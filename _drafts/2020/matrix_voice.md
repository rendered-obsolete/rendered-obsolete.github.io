---
layout: post
title: Matrix Voice
tags:
- smarthome
- iot
- homeautomation
series: Smart Home
---

## Matrix Voice


- [Matrix Voice (w/ ESP32)](https://matrix-io.github.io/matrix-documentation/matrix-voice/overview/)
- Raspberry Pi 3 ([or other compatible host](https://matrix-io.github.io/matrix-documentation/matrix-voice/device-setup/))
    - Micro SD card with Rasbian
    - 5V 2.5A power supply


1. Connect Matrix Voice to Raspberry Pi.
1. Follow the [setup instructions](https://matrix-io.github.io/matrix-documentation/matrix-voice/esp32/):
```sh
# Ssh into the Pi
pc> ssh pi@pi3.local

# Install software
curl https://apt.matrix.one/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.matrix.one/raspbian $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/matrixlabs.list
sudo apt-get update
sudo apt-get upgrade
sudo apt install matrixio-creator-init
sudo reboot

# Wait for Pi to reboot and ssh back in
pc> ssh pi@pi3.local

# Enable ESP32 communication and reset ESP32 flash memory
sudo voice_esp32_enable
esptool.py --chip esp32 --port /dev/ttyS0 --baud 115200 --before default_reset --after hard_reset erase_flash
```

Output:
```
esptool.py v2.8
Serial port /dev/ttyS0
Connecting....
Chip is ESP32D0WDQ6 (revision 1)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
WARNING: Detected crystal freq 41.01MHz is quite different to normalized freq 40MHz. Unsupported crystal in use?
Crystal is 40MHz
MAC: 80:7d:3a:ee:77:58
Uploading stub...
Running stub...
Stub running...
Erasing flash (this may take a while)...
Chip erase completed successfully in 14.0s
Hard resetting via RTS pin...
```

## PC

1. Install [PlatformIO](https://platformio.org/) for your IDE.  E.g. for VSCode:
    1. Open Extension manager
    1. Search for `PlatformIO`
1. [pio to PATH](https://docs.platformio.org/en/latest/installation.html#install-shell-commands)

1. `git clone https://github.com/matrix-io/esp32-platformio`

[Troubleshooting](https://matrix-io.github.io/matrix-documentation/matrix-voice/troubleshooting/)
```sh
sudo apt-get --purge remove matrixio-creator-init
sudo reboot

pc> ssh pi@pi3.local
sudo apt-get install matrixio-creator-init
sudo reboot
```

https://community.matrix.one/t/solved-unable-to-get-sensor-data-of-matrix-creator-on-rpi-3b/1666

If it isn't a Matrix Voice with an ESP32:
```
A fatal error occurred: Failed to connect to Espressif device: Invalid head of packet (’\x00’)
```

Try re-running `./install.sh`:
```
Wrote 966656 bytes at 0x00010000 in 11.7 seconds (661.1 kbit/s)...
File  md5: 3740260ea2f89a34fa43c992c41f3418
Flash md5: 3740260ea2f89a0034fa43c992c41f18
MD5 of 0xFF is e0d20479298405805af99b0c64266c4e

A fatal error occurred: MD5 of file does not match data in flash!
```

## SDK

https://rust-embedded.github.io/book/intro/install/macos.html
https://matrix-io.github.io/matrix-documentation/

## Naked

https://www.hackster.io/matrix-labs/program-over-the-air-on-esp32-matrix-voice-w-arduino-ide-5e76bb

https://docs.espressif.com/projects/esp-idf/en/latest/hw-reference/get-started-wrover-kit.html