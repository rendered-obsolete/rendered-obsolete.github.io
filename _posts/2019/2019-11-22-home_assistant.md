---
layout: post
title: Installing Home Assistant
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

For quite some time now I've been plotting to put a [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) to work as some kind of home automation/"smart" home thingy.  After looking at a few options, I've more or less settled on [Home Assistant](https://www.home-assistant.io/)- aka "Hass".  It seems to check all the boxes: [open-source](https://github.com/home-assistant/home-assistant), optionally offline, just-works™, and not fugly.  An official Android app was also [recently announced](https://www.home-assistant.io/blog/2019/11/20/release-102/).

Before getting started, [install Raspbian]({% post_url /2019/2019-03-21-raspi_3 %}) or some other distribution of your choice.

## Install

Installing Hass is easy.  The top-level [install instructions](https://www.home-assistant.io/docs/installation/) list three ways to do it:  
- [Hass.io](https://www.home-assistant.io/hassio/); a full system image you would run as an appliance or in a VM
- [Docker](https://www.home-assistant.io/docs/installation/docker/)
- [Manually in Python virtual environment](https://www.home-assistant.io/docs/installation/virtualenv)

Here we'll go through the manual route also referencing [additional instructions for the Raspberry Pi](https://www.home-assistant.io/docs/installation/raspberry-pi/).  If you're unfamiliar with Python/pip/venv, read ["Installing packages using pip and virtual environments"](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/).

If on Stretch need to manually install python >=3.7; apt only gives you 3.5:
```sh
# Build python 3.7
# `--enable-optimizations` uses tests that take a long time.  First do a quick install and then do an optimized build later.
./configure --with-ensurepip=install
make -j2
make altinstall

sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.5 50
sudo update-alternatives --install /usr/bin/python3 python3 /usr/local/bin/python3.7 40
# Switch to python3.7
sudo update-alternatives --set python3 /usr/local/bin/python3.7
# Or, for the TUI to manually select
#sudo update-alternatives --config python3
```

```sh
# Install pre-requisites
sudo apt-get install -y python3 python3-dev python3-venv python3-pip libffi-dev libssl-dev

# Create `homeassistant` user Hass will run as
sudo useradd -rm homeassistant -G dialout,gpio,i2c,audio
cd /srv
sudo mkdir homeassistant
sudo chown homeassistant:homeassistant homeassistant
# Switch to `homeassistant` user
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

The first time it runs a bunch of additional packages will be installed.  Once the following text appears installation has completed:
```
INFO (MainThread) [homeassistant.core] Timer:starting
```

## Auto Start

[Additional instructions](https://www.home-assistant.io/docs/autostart/) are provided to make Hass start automatically at boot.

As super-user create `/etc/systemd/system/home-assistant@YOUR_USER.service` (where `YOUR_USER` is the name of the user you created above, in this case `homeassistant`):
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

For example:
```sh
sudo vim /etc/systemd/system/home-assistant@homeassistant.service
## Paste the above configuration then `:wq`

# Reload systemd and enable to start automatically at boot
sudo systemctl --system daemon-reload
sudo systemctl enable home-assistant@homeassistant
# Start it now
sudo systemctl start home-assistant@homeassistant
```

For help debugging Hass installs:
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

My Pi is tucked inside the official [7" Touch Display](https://www.raspberrypi.org/products/raspberry-pi-touch-display/), and I'd like to have it launch the Hass GUI as a dashboard to serve as a control panel for my entire flat.  The two main approaches are:  
- [systemd](#systemd)
- [autostart](#autostart)

There's a few more ideas in [this forum thread](https://www.raspberrypi.org/forums/viewtopic.php?t=8298).

### Systemd

I first tried the solution from [this post](https://jonathanmh.com/raspberry-pi-4-kiosk-wall-display-dashboard/) about starting a dashboard on a Raspberry Pi 4.  It didn't work for me, but I didn't spend much time debugging it so I plan to come back to it.

As super-user create `/etc/systemd/system/dashboard.service`:
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

Create `~/dashboard.sh`:
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

### Autostart

The solution that did work for me was following a [sparkfun tutorial](https://learn.sparkfun.com/tutorials/how-to-run-a-raspberry-pi-program-on-startup/all) covers using autostart to launch a program at startup.  Create `~/.config/autostart/dashboard.desktop`:
```ini
[Desktop Entry]
Type=Application
Name=Dashboard
Exec=/home/pi/dashboard.sh
```

Here using the same `dashboard.sh` from above.

### End Result 

The default GUI is servicable:  
![](/assets/raspi3_hass.jpg)

During Hass initial setup it automatically detected the [Philips Hue](https://www2.meethue.com/en-us) "smart lighting" in my home.  You can turn lights on and off, adjust the brightness, etc.  And by supplying my location there's a weather widget with the local weather and forecast.

This serves as a good starting point.  Next I'll be looking at doing some actual automation and getting some other features working.

## Common Problems

If you don't install ffi, `pip install homeassistant` will fail with:
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

If you ssh in without `-X`, `hass --open-ui` gets stuck without UI:
```
2019-10-04 22:02:40 INFO (MainThread) [homeassistant.core] Timer:starting
```
