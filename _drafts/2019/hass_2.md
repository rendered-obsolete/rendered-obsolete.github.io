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
