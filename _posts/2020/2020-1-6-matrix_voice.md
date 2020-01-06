---
layout: post
title: Matrix Voice- A Microphone Array w/ Microcontroller
tags:
- smarthome
- iot
- homeautomation
series: Smart Home
---

While looking at microphone arrays like the [ReSpeaker]({% post_url /2019/2019-12-10-respeaker %}) and thinking of different pairings such as with one of the [Pi Zeros I've already got]({% post_url /2019/2019-03-28-raspi_zero %}) or a simpler microcontroller (MCU), I came across an interesting product, the [Matrix Voice](https://www.matrix.one/products/voice)- a microphone array with an (optional) MCU:

- FPGA: Xilinx Spartan 6 XC6SLX9
- 8 MEMS microphones
- RAM: 64 MByte 132MHz SDRAM
- FLASH: 64 Mbit Flash Memory
- MCU: [ESP32](https://en.wikipedia.org/wiki/ESP32) (Espressif [ESP32-WROOM-32](https://www.espressif.com/en/products/hardware/esp-wroom-32/overview))
    - CPU: Tensilica Xtensa dual-core 32-bit LX6 at 240MHz
    - Wifi: 802.11bgn 2.4GHz
    - BT: 4.2
    - RAM: 520 KByte
    - Flash: 4 MByte

Decided it was interesting enough to warrant a look. The following notes are based off the [official Matrix tutorial](https://matrix-io.github.io/matrix-documentation/matrix-voice/esp32/) ([also on Hackster.io](https://www.hackster.io/matrix-labs/program-matrix-voice-esp32-with-vs-code-using-platformio-3dd498
) with annoying registration pop-up).


## Requirements

- [Matrix Voice (w/ ESP32)](https://matrix-io.github.io/matrix-documentation/matrix-voice/overview/)
    - Note there is a version _without_ ESP32 MCU
- Raspberry Pi or [other supported device](https://matrix-io.github.io/matrix-documentation/matrix-voice/device-setup/)
    - Micro SD card [with Debian Buster]({% post_url /2019/2019-03-21-raspi_3 %})
    - 5V 2.5A power supply

## Pi Setup

1. Turn off the Pi, connect Matrix Voice to Pi's GPIO pins, and turn the Pi back on
1. Follow the "Raspberry Pi Setup" instructions:  

```sh
# On PC: ssh into the Pi
ssh pi@raspberrypi.local

# Add Matrix key and repo
curl https://apt.matrix.one/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.matrix.one/raspbian $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/matrixlabs.list
# Update repositories and packages
sudo apt-get update
sudo apt-get upgrade -y # This may take a while

# Matrix Voice init and restart
sudo apt install -y matrixio-creator-init
sudo reboot

# On PC: Wait for Pi to reboot and ssh back in
ssh pi@raspberrypi.local

# Enable ESP32 communication
sudo voice_esp32_enable
# Reset ESP32 flash memory
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

## PC Setup

1. Install [PlatformIO](https://platformio.org/) for your environment/IDE.  E.g. for VSCode:
    1. Open Extension manager
    1. Search for `PlatformIO`
    1. __Install__
1. Add [pio to PATH](https://docs.platformio.org/en/latest/installation.html#install-shell-commands)
1. Git [Matrix PlatformIO project](https://github.com/matrix-io/esp32-platformio)
1. In `esp32-platformio/platformio.ini` change `SSID_GOES_HERE` and `PASSWORD_GOES_HERE` to the SSID and password of your wifi

For Mac/Linux:
```sh
# Add pio to PATH
export PATH=$PATH:~/.platformio/penv/bin
# Git Matrix PlatformIO project
git clone https://github.com/matrix-io/esp32-platformio
```

## Build and Deploy

```sh
# Build
cd esp32-platformio/
pio run
# Deploy
cd ota/
./install.sh raspberrypi.local
```

At this point you should be "rewarded" with a nauseating LED demo.  But, mine did nothing...

## Troubleshooting

First off, [if it doesn't have an ESP32](https://community.matrix.one/t/solved-unable-to-get-sensor-data-of-matrix-creator-on-rpi-3b/):
```
A fatal error occurred: Failed to connect to Espressif device: Invalid head of packet (’\x00’)
```

If you get any of the following, try [re-deploying with `install.sh`](#build-and-deploy):
```
A fatal error occurred: MD5 of file does not match data in flash!
A fatal error occurred: MD5Sum command returned unexpected result:
A fatal error occurred: Timed out waiting for packet header
A fatal error occurred: Invalid head of packet (0x80)
```

Otherwise, follow their excellent [troubleshooting](https://matrix-io.github.io/matrix-documentation/matrix-voice/troubleshooting/) guide.

Reinstall `matrixio-creator-init` package:
```sh
# On Pi
sudo apt-get --purge remove matrixio-creator-init
sudo reboot

# On PC: ssh into pi
ssh pi@raspberrypi.local
sudo apt-get install matrixio-creator-init
sudo reboot
```

Reflash the FPGA.  The `blob/*.bit` files you pass to `xc3sprog` are installed by `matrix-creator-init` package in `/usr/share/matrixlabs/matrixio-devices/`:
```sh
# Reset the FPGA
echo 26 > /sys/class/gpio/export 2>/dev/null
echo out > /sys/class/gpio/gpio26/direction  
echo 1 > /sys/class/gpio/gpio26/value  
echo 0 > /sys/class/gpio/gpio26/value  
echo 1 > /sys/class/gpio/gpio26/value
# Flash the SPI Flash bootloader onto FPGA
xc3sprog -c matrix_voice /usr/share/matrixlabs/matrixio-devices/blob/bscan_spi_s6lx9_ftg256.bit
# Flash the SPI Flash
xc3sprog -c matrix_voice -I /usr/share/matrixlabs/matrixio-devices/blob/system_voice.bit
# Reset the FPGA
echo 26 > /sys/class/gpio/export 2>/dev/null
echo out > /sys/class/gpio/gpio26/direction  
echo 1 > /sys/class/gpio/gpio26/value  
echo 0 > /sys/class/gpio/gpio26/value  
echo 1 > /sys/class/gpio/gpio26/value
# Shutdown the pi
sudo shutdown
# Unplug power, plug it back in, turn power on
```

Run hardware tests:
```sh
# Matrix init script
sudo /usr/share/matrixlabs/matrixio-devices/matrix-init.bash
# Should output:
#*** MATRIX Voice has a updated firmware
#*** MATRIX Voice initial process has been launched

# Check FPGA info
sudo /usr/share/matrixlabs/matrixio-devices/fpga_info
```

None of this worked for me, but the next thing did.

### Serial Output

While Matrix Voice is connected to the Pi, you can read serial output from the ESP32.  If you're [already using screen]({% post_url /2019/2019-03-21-raspi_3 %}#screen):
```sh
# On the pi
screen /dev/ttyS0 115200
```

Log spam ahoy- it's stuck in a reboot loop:
```
I (12) boot: ESP-IDF v4.0-dev-459-gba1ff1692 2nd stage bootloader
I (12) boot: compile time 11:07:17
I (12) boot: Enabling RNG early entropy source...
I (17) boot: SPI Speed      : 40MHz
I (21) boot: SPI Mode       : DIO
I (25) boot: SPI Flash Size : 4MB
I (29) boot: Partition Table:
I (33) boot: ## Label            Usage          Type ST Offset   Length
I (40) boot:  0 nvs              WiFi data        01 02 00009000 00004000
I (47) boot:  1 otadata          OTA data         01 00 0000d000 00002000
I (55) boot:  2 phy_init         RF data          01 01 0000f000 00001000
I (62) boot:  3 factory          factory app      00 00 00010000 00100000
I (70) boot:  4 ota_0            OTA app          00 10 00110000 00100000
I (77) boot:  5 ota_1            OTA app          00 11 00210000 00100000
I (85) boot: End of partition table
I (89) boot: Defaulting to factory image
I (94) esp_image: segment 0: paddr=0x00010020 vaddr=0x3f400020 size=0x31c20 (203808) map
I (174) esp_image: segment 1: paddr=0x00041c48 vaddr=0x3ffbdb60 size=0x03558 ( 13656) load
I (180) esp_image: segment 2: paddr=0x000451a8 vaddr=0x40080000 size=0x00400 (  1024) load
I (181) esp_image: segment 3: paddr=0x000455b0 vaddr=0x40080400 size=0x0aa60 ( 43616) load
I (208) esp_image: segment 4: paddr=0x00050018 vaddr=0x400d0018 size=0xa0d84 (658820) map
I (439) esp_image: segment 5: paddr=0x000f0da4 vaddr=0x4008ae60 size=0x09574 ( 38260) load
I (467) boot: Loaded app from partition at offset 0x10000
I (467) boot: Disabling RNG early entropy source...
Guru Meditation Error: Core  0 panic'ed (LoadProhibited). Exception was unhandled.
Core 0 register dump:
PC      : 0x4016db0b  PS      : 0x00060730  A0      : 0x8012c27f  A1      : 0x3ffe3b00  
A2      : 0x00000000  A3      : 0x000000fe  A4      : 0x00000001  A5      : 0x00000000  
A6      : 0x00000000  A7      : 0x00000000  A8      : 0x8012bda5  A9      : 0x3ffe3ad0  
A10     : 0x00000020  A11     : 0x4012b960  A12     : 0x3ffe3ae8  A13     : 0x00000000  
A14     : 0x00000000  A15     : 0x00000000  SAR     : 0x0000001b  EXCCAUSE: 0x0000001c  
EXCVADDR: 0x00000000  LBEG    : 0x4000c46c  LEND    : 0x4000c477  LCOUNT  : 0x00000000  

Backtrace: 0x4016db0b:0x3ffe3b00 0x4012c27c:0x3ffe3b20 0x400d1c01:0x3ffe3b50 0x400d147e:0x3ffe3ba0 0x400d5b2b:0x3ffe3bc0 0x40081c7d:0x3ffe3be0 0x40081e8c:0x3ffe3c10 0x400792f7:0x3ffe3c30 0x400793a9:0x3ffe3c60 0x400793c7:0x3ffe3ca0 0x400796ad:0x3ffe3cc0 0x40080796:0x3ffe3df0 0x40007c31:0x3ffe3eb0 0x4000073d:0x3ffe3f20

Rebooting...
```

Which led me to the solution in [this forum post](https://community.matrix.one/t/matrix-voice-esp32-constant-rebooting).  In `platformio.ini` change:
```ini
# OLD
#platform = espressif32
# NEW
platform = espressif32@1.9.0
```

It wasn't necessary in my case, but you may also need to:
```sh
pio update
pio lib update
```

## OTA

At this point the LEDs should be running wild.  The Matrix Voice should also be connected to your wifi.  You can either check the connected device list on your AP/router, or you can try pinging `MVESP.local` (the hostname specified in `platformio.ini`).

You should also be able to push an [over-the-air (OTA)](https://en.wikipedia.org/wiki/Over-the-air_programming) update to the Matrix Voice even when it is not connected to the Pi:

1. Turn off the pi: `sudo shutdown`
1. Disconnect Matrix Voice from Pi
1. Connect power to Matrix Voice's micro USB
    - After ~2 seconds the LEDs should restart
1. On PC:
```sh
cd esp32-platformio/
pio run --target upload # Uses settings in platformio.ini
```
