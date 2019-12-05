---
layout: post
title: Respeaker
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

Hardware:  

- [ReSpeaker USB 4 Mic Array](https://respeaker.io/usb_4_mic_array/)
- [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)

## Basic Sound

I never did much multi-media work with Linux, and I really struggled.

Abridged `arecord --help` output:
```
Usage: arecord [OPTION]... [FILE]...

-l, --list-devices      list all soundcards and digital audio devices
-L, --list-pcms         list device names
-D, --device=NAME       select PCM by name
-t, --file-type TYPE    file type (voc, wav, raw or au)
-c, --channels=#        channels
-f, --format=FORMAT     sample format (case insensitive)
-r, --rate=#            sample rate
-d, --duration=#        interrupt after # seconds
-v, --verbose           show PCM structure and setup (accumulative)
```

List various devices.  `arecord -l`:
```
**** List of CAPTURE Hardware Devices ****
card 1: Dummy [Dummy], device 0: Dummy PCM [Dummy PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 2: ArrayUAC10 [ReSpeaker 4 Mic Array (UAC1.0)], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```
Then, `arecord -L`:
```
null
    Discard all samples (playback) or generate zero samples (capture)
default
<Bunch of CARD=Dummy>
sysdefault:CARD=ArrayUAC10
    ReSpeaker 4 Mic Array (UAC1.0), USB Audio
    Default Audio Device
<Bunch of CARD=ArrayUAC10 speakers/output>
dmix:CARD=ArrayUAC10,DEV=0
    ReSpeaker 4 Mic Array (UAC1.0), USB Audio
    Direct sample mixing device
dsnoop:CARD=ArrayUAC10,DEV=0
    ReSpeaker 4 Mic Array (UAC1.0), USB Audio
    Direct sample snooping device
hw:CARD=ArrayUAC10,DEV=0
    ReSpeaker 4 Mic Array (UAC1.0), USB Audio
    Direct hardware device without any conversions
plughw:CARD=ArrayUAC10,DEV=0
    ReSpeaker 4 Mic Array (UAC1.0), USB Audio
    Hardware device with all software conversions
```

## Voice Recognition

https://github.com/snipsco/snips-issues/issues/161
https://docs.snips.ai/getting-started/quick-start-jetson-tx2
https://docs.snips.ai/getting-started
https://docs.snips.ai/articles/other-platforms


https://www.home-assistant.io/integrations/snips

https://docs.snips.ai/articles/raspberrypi/manual-setup
```sh
sudo apt-get install -y dirmngr
sudo bash -c  'echo "deb https://raspbian.snips.ai/$(lsb_release -cs) stable main" > /etc/apt/sources.list.d/snips.list'
sudo apt-key adv --fetch-keys https://raspbian.snips.ai/531DD1A7B702B14D.pub
sudo apt-get update
sudo apt-get install -y snips-platform-voice
```

https://chrisjean.com/fix-apt-get-update-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available/
```sh
sudo apt-key adv --keyserver pgp.mit.edu --recv-keys D4F50CDCA10A2849
```

1. Click __Add an App__
1. Click `+` of apps of interest
1. Click __Add Apps__ button
1. Wait for training to complete
1. Click __Deploy Assistant__ button
1. __Download and install manually__

https://docs.snips.ai/articles/console/actions/deploy-your-assistant#deploy-your-assistant-manually-without-sam

https://docs.snips.ai/getting-started/quick-start-raspberry-pi#step-3-install-the-snips-platform



```sh
sudo unzip assistant_proj_XYZ.zip -d /usr/share/snips
sudo systemctl restart 'snips-*'

sudo apt-get install -y mosquitto-clients
mosquitto_sub -p 1883 -t "#"

tail -f /var/log/syslog
```

https://stackoverflow.com/questions/20760589/list-all-audio-devices-with-pythons-pyaudio-portaudio-binding
```py
import pyaudio
p = pyaudio.PyAudio()
for i in range(p.get_device_count()):
    print p.get_device_info_by_index(i)
```
