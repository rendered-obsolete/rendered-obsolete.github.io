---
layout: post
title: 
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

##


Update containers ([SO](https://stackoverflow.com/questions/49316462)):
```sh
docker-compose up --force-recreate --build -d
docker image prune -f
```

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


https://www.home-assistant.io/docs/mqtt/broker/
https://www.home-assistant.io/docs/configuration/
vim ~/.homeassistant/configuration.yaml
hass --script check_config


https://docs.snips.ai/getting-started/quick-start-jetson-tx2
https://docs.snips.ai/getting-started
https://docs.snips.ai/articles/other-platforms





https://docs.snips.ai/getting-started/quick-start-raspberry-pi#step-3-install-the-snips-platform

https://docs.snips.ai/guides/advanced-configuration/wakeword/personal-wakeword



https://chrisjean.com/fix-apt-get-update-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available/
```sh
sudo apt-key adv --keyserver pgp.mit.edu --recv-keys D4F50CDCA10A2849
```
