---
layout: post
title: Home Automation with Home Assistant
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

I've been plotting to put a [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) to work as some kind of "smart" home something.

[Install Rasbian]({% post_url /2019/2019-03-21-raspi_3 %})

https://github.com/home-assistant/home-assistant

[Hass.io](https://www.home-assistant.io/hassio/) which is solution for a hub.

## Install

Installing Hass is easy.  There's [general instructions](https://www.home-assistant.io/docs/installation/) as well as [instructions specific to the Raspberry Pi](https://www.home-assistant.io/docs/installation/raspberry-pi/).  In either case, they recommend installing into a [Python virtual environment](https://www.home-assistant.io/docs/installation/virtualenv):  
```sh
# Install pre-requisites
sudo apt-get install -y python3 python3-dev python3-venv python3-pip libffi-dev libssl-dev

# Create `homeassistant` user
sudo useradd -rm homeassistant -G dialout,gpio,i2c,audio
cd /srv
sudo mkdir homeassistant
sudo chown homeassistant:homeassistant homeassistant
sudo -u homeassistant -H -s

# Create python virtual environment and switch to it
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

If you're unfamiliar with Python or venv, read ["Installing packages using pip and virtual environments"](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/).

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

My Pi is tucked inside the official [7" Touch Display](https://www.raspberrypi.org/products/raspberry-pi-touch-display/), and I'd like to have it auto-start the Hass GUI as a dashboard to serve as a control panel for my entire flat.

### Systemd Service

[This post](https://jonathanmh.com/raspberry-pi-4-kiosk-wall-display-dashboard/) covers one approach to starting a dashboard on Raspberry Pi 4.  `sudo vim /etc/systemd/system/dashboard.service` and add:
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

```sh
# Make the script executable
chmod +x ~/dashboard.sh
# Install helper
sudo apt-get install unclutter
# Enable the service
sudo systemctl enable dashboard.service
# Optionally, view log output
sudo journalctl -f -u dashboard.service
```

This didn't work for me, but I didn't spend much time debugging it.

### Autostart

A [sparkfun tutorial](https://learn.sparkfun.com/tutorials/how-to-run-a-raspberry-pi-program-on-startup/all) details an easier way to launch a program at startup.  In `~/.config/autostart/dashboard.desktop`:
```ini
[Desktop Entry]
Type=Application
Name=Dashboard
Exec=/home/pi/dashboard.sh
```

Here we're using the same `dashboard.sh` from above.

Some more ideas in [this forum thread](https://www.raspberrypi.org/forums/viewtopic.php?t=8298).

### End Result 

The default GUI is servicable:  
![](/assets/raspi3_hass.jpg)

You can turn lights on and off, adjust the brightness, etc.  And there's a weather widget.

## Common Problems

If you don't install ffi `pip install homeassistant` will fail with:
```sh
      c/_cffi_backend.c:15:10: fatal error: ffi.h: No such file or directory
       #include <ffi.h>
                ^~~~~~~
```

There are [issues](https://github.com/home-assistant/home-assistant/issues/15720) installing hass without venv:
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

If not using venv and `python3 -m pip install` without `sudo` or `--user`:
```
ERROR: Could not install packages due to an EnvironmentError: [Errno 13] Permission denied: '/usr/local/lib/python3.6/dist-packages/_cffi_backend.cpython-36m-aarch64-linux-gnu.so'
Consider using the `--user` option or check the permissions.
```

If you ssh in without `-X`, `hass --open-ui` gets stuck:
```
2019-10-04 22:02:40 INFO (MainThread) [homeassistant.core] Timer:starting
```

Oct 12 10:22:50 jake-pi3 dhcpcd[379]: wlan0: soliciting a DHCPv6 lease
Oct 12 10:22:50 jake-pi3 dhcpcd[379]: wlan0: dropping DHCPv6 due to no valid routers