---
layout: post
title: Home Automation with Home Assistant
tags:
- hass
- iot
canonical_url:
---


https://github.com/home-assistant/home-assistant

Hass.io which is solution for a hub.

## Install

Compared to jasper Hass is a breeze.
https://www.home-assistant.io/docs/installation/
https://www.home-assistant.io/docs/installation/raspberry-pi/

[Python virtual environment](https://www.home-assistant.io/docs/installation/virtualenv)

```sh
# Install pre-requisites
sudo apt-get install -y python3 python3-dev python3-venv python3-pip libffi-dev libssl-dev

# Create `homeassistant` user and python virtual environment
sudo useradd -rm homeassistant -G dialout,gpio,i2c,audio
cd /srv
sudo mkdir homeassistant
sudo chown homeassistant:homeassistant homeassistant
sudo -u homeassistant -H -s
cd /srv/homeassistant
python3 -m venv .
source bin/activate

# Install Home Assistant in venv and run once to complete setup
venv$ python3 -m pip install wheel
venv$ pip3 install homeassistant
venv$ hass
```

Completed once the following text is output:
```
INFO (MainThread) [homeassistant.core] Timer:starting
```

## Auto Start

https://www.home-assistant.io/docs/autostart/

/etc/systemd/system/home-assistant@YOUR_USER.service

`sudo vim /etc/systemd/system/home-assistant@homeassistant.service`
```ini
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
ExecStart=/srv/homeassistant/bin/hass -c "/home/%i/.homeassistant"
# Restart on failure
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```sh
# Reload systemd and enable to start automatically at boot
sudo systemctl --system daemon-reload
sudo systemctl enable home-assistant@homeassistant
# Start it now
sudo systemctl start home-assistant@homeassistant
```

Debugging
```sh
sudo systemctl status home-assistant@homeassistant
# Stop Home Assistant
sudo systemctl stop home-assistant@homeassistant
# Restart Home Assistant
sudo systemctl restart home-assistant@homeassistant
# View log output
sudo journalctl -f -u home-assistant@homeassistant
```

## Dashboard
https://jonathanmh.com/raspberry-pi-4-kiosk-wall-display-dashboard/

`sudo vim /etc/systemd/system/dashboard.service` and add:
```ini
[Unit]
Description=Chromium Dashboard
Requires=graphical.target
After=graphical.target

[Service]
Environment=DISPLAY=:0.0
Environment=XAUTHORITY=/home/pi/.Xauthority
Type=simple
ExecStart=/home/pi/dashboard.sh
Restart=on-abort
User=pi
Group=pi

[Install]
WantedBy=graphical.target
```

```sh
sudo systemctl enable dashboard.service
```

`sudo apt-get install unclutter`

`vim ~/dashboard.sh` and add:
```sh
#!/usr/bin/env bash

# Prevent putting screen to sleep
xset s noblank
xset s off
xset -dpms

# Hide mouse cursor
unclutter -idle 0.5 -root &

# Ignore "unclean" Chrome shutdowns
sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' /home/pi/.config/chromium/Default/Preferences
sed -i 's/"exit_type":"Crashed"/"exit_type":"Normal"/' /home/pi/.config/chromium/Default/Preferences

# Open Home Assistant dashboard
/usr/bin/chromium-browser --noerrdialogs --disable-infobars --kiosk http://localhost:8123 &
```

`sudo chmod +x ~/dashboard.sh`

```sh
sudo journalctl -f -u dashboard.service
```

https://learn.sparkfun.com/tutorials/how-to-run-a-raspberry-pi-program-on-startup/all

In `~/.config/autostart/dashboard.desktop`:
```ini
[Desktop Entry]
Type=Application
Name=Dashboard
Exec=/home/pi/dashboard.sh
```

Some more ideas in [this forum thread](https://www.raspberrypi.org/forums/viewtopic.php?t=8298).

## Media Player

[Kodi](https://www.raspberrypi.org/documentation/usage/kodi/README.md)
https://kodi.wiki/view/HOW-TO:Install_Kodi_on_Raspberry_Pi
https://www.home-assistant.io/integrations/kodi

```sh
sudo apt-get install -y kodi
# Switch to `homeassistant` user
sudo -u homeassistant -H -s
```

Add the following to `/home/homeassistant/.homeassistant/configuration.yaml`:
```yaml
media_player:
  - platform: vlc
```

```sh
# Enter venv
source /srv/homeassistant/bin/activate
# Validate configuration.yaml
venv$ hass --script check_config
# Restart Home Assistant
sudo systemctl restart home-assistant@homeassistant
```

## Voice Recognition

https://github.com/snipsco/snips-issues/issues/161

https://www.home-assistant.io/integrations/snips
https://docs.snips.ai/getting-started/quick-start-jetson-tx2
https://docs.snips.ai/getting-started
https://docs.snips.ai/articles/raspberrypi/manual-setup
https://docs.snips.ai/articles/other-platforms

https://chrisjean.com/fix-apt-get-update-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available/
```sh
sudo apt-key adv --keyserver pgp.mit.edu --recv-keys D4F50CDCA10A2849

sudo apt-get install -y snips-platform-voice
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

https://www.home-assistant.io/docs/mqtt/broker/
https://www.home-assistant.io/docs/configuration/
vim ~/.homeassistant/configuration.yaml
hass --script check_config


## Common Problems

If you don't install ffi `pip install homeassistant` will fail with:
```sh
      c/_cffi_backend.c:15:10: fatal error: ffi.h: No such file or directory
       #include <ffi.h>
                ^~~~~~~
```

If you `python3 -m pip install` without `sudo` or `--user`:
```
ERROR: Could not install packages due to an EnvironmentError: [Errno 13] Permission denied: '/usr/local/lib/python3.6/dist-packages/_cffi_backend.cpython-36m-aarch64-linux-gnu.so'
Consider using the `--user` option or check the permissions.
```

https://github.com/home-assistant/home-assistant/issues/15720
```
2019-10-04 03:16:59 INFO (MainThread) [homeassistant.setup] Setting up recorder
Exception in thread Recorder:
Traceback (most recent call last):
  File "/usr/lib/python3.6/threading.py", line 916, in _bootstrap_inner
    self.run()
  File "/usr/local/lib/python3.6/dist-packages/homeassistant/components/recorder/__init__.py", line 211, in run
    from .models import States, Events
  File "/usr/local/lib/python3.6/dist-packages/homeassistant/components/recorder/models.py", line 6, in <module>
    from sqlalchemy import (
ModuleNotFoundError: No module named 'sqlalchemy'

2019-10-04 03:17:09 WARNING (MainThread) [homeassistant.setup] Setup of recorder is taking over 10 seconds.
```
https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/

If you ssh in and `hass --open-ui` gets stuck
```
2019-10-04 22:02:40 INFO (MainThread) [homeassistant.core] Timer:starting
```

Oct 12 10:22:50 jake-pi3 dhcpcd[379]: wlan0: soliciting a DHCPv6 lease
Oct 12 10:22:50 jake-pi3 dhcpcd[379]: wlan0: dropping DHCPv6 due to no valid routers