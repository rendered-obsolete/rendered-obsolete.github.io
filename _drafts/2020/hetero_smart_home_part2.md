---
layout: post
title: Heterogeneous Smart Home (Part 2)
tags:
- smarthome
- iot
- homeautomation
- homeassistant
series: Smart Home
---

Both Zigbee and Z-Wave

https://phoscon.de/en/conbee2

- Integration with Home Assistant via [deCONZ](https://www.home-assistant.io/integrations/deconz)
- Compatible with numerous devices from Philips, IKEA, Xiaomi, etc. ([list](https://phoscon.de/en/conbee2/compatible))
- [Completely offline](https://phoscon.de/en/conbee2/#nocloud)


[docker image](https://phoscon.de/en/conbee2/install#docker)

`docker-compose.yml`
https://github.com/marthoc/docker-deconz#docker-compose
```yml
services:
  deconz:
    image: marthoc/deconz:stable
    container_name: deconz
    network_mode: host
    restart: unless-stopped
    volumes:
      - /opt/deconz:/root/.local/share/dresden-elektronik/deCONZ
    ports:
      - "80:80"
      - "443:443"
    devices:
      - /dev/ttyUSB0
    environment:
      - DECONZ_WEB_PORT=80
      - DECONZ_WS_PORT=443
      - DEBUG_INFO=1
      - DEBUG_APS=0
      - DEBUG_ZCL=0
      - DEBUG_ZDP=0
      - DEBUG_OTAU=0
```

```sh
mkdir -p ~/.local/share/dresden-elektronik/deCONZ
```