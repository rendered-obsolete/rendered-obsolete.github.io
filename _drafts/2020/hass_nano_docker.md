---
layout: post
title: Addendum- Migrating to Jetson Nano and Docker
tags:
- smarthome
- iot
- homeautomation
- hass
- docker
series: Smart Home
---

After [installing Rhasspy on a Raspberry Pi 3B]({% post_url /2020/2020-1-2-rhasspy %}) and seeing it struggle, I decided to move everything to a beefier [Jetson Nano]({% post_url /2019/2019-04-01-jetson_nano %}).  While I was at it, took the opportunity to switch to docker to avoid some of the [systemd wrangling]({% post_url /2020/2020-1-2-rhasspy %}#systemd).

## Rhasspy

Rhasspy provides speech recognition, speech-to-intent, and text-to-speech.  Installation is mostly the same as before:
```sh
docker run -d -p 12101:12101 \
      --name rhasspy \
      --restart unless-stopped \
      -v "$HOME/.config/rhasspy/profiles:/profiles" \
      --device /dev/snd:/dev/snd \
      --device /dev/bus/usb:/dev/bus/usb \
      synesthesiam/rhasspy-server:latest \
      --user-profiles /profiles \
      --profile en
```

With the Nano, I needed to add `--device /dev/bus/usb:/dev/bus/usb` so USB devices like [ReSpeaker]({% post_url /2019/2019-12-10-respeaker %}) are also accessible inside the container.

## Home Assistant

Home Assistant (Hass) is the "smart home" platform and among other things provides: intent handling, automation, location-based services, etc.  Hass has [docker installation instructions](https://www.home-assistant.io/docs/installation/docker/):  
```sh
docker run --init -d --name="home-assistant" -e "TZ=America/New_York" -v /PATH_TO_YOUR_CONFIG:/config --net=host homeassistant/home-assistant:stable
```

We'll change this to:
```sh
docker run -d -p 8123:8123 \
    --name hass \
    --restart unless-stopped \
    -v $HOME/.homeassistant:/config \
    -e "TZ=Asia/Nicosia" \
    homeassistant/home-assistant:stable
```

- Shorten the name to "hass", just because typing "home-assistant" is tedious
- `unless-stopped` [restart policy](https://docs.docker.com/engine/reference/run/#restart-policies---restart)- the docker daemon will automatically (re-)start it
- `$HOME/.homeassistant` like when doing manual install
- Value after `TZ=` with the appropriate name from ["tz database" time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
- `host` networking on Linux means we don't need to forward any [ports for the frontend]({% post_url /2019/2019-11-22-home_assistant %}) like `-p 8123:8123`, but doesn't seem to work with `docker network` ([see below](#integration))


## Integration

Both Hass and Rhasspy are running in containers, but they can't talk to eachother yet.  The "old" way to connect two containers was to use [`--link`](https://docs.docker.com/network/links/).  The new way is using ["networks"](https://docs.docker.com/engine/reference/commandline/network_connect/).  First, create a network called "smart-home":  
```sh
docker network create smart-home
```
To each of the `docker run` commands we then add `--network smart-home`.  For Hass, we also remove `--net=host` and add `-p 8123:8123` (for the web frontend).  Restart the containers and we can confirm that Rhasspy can talk to the "hass" container:  
```sh
docker exec -it rhasspy bash
# Install ping (or some other command)
apt install -y inetutils-ping
ping hass
```

Now, when [connecting Rhasspy to Hass]({% post_url /2020/2020-1-2-rhasspy %}#hass-integration), for __Hass URL__ use `http://hass:8123`.

## Start-Up

With everything on docker we don't really need systemd to start the individual services, just the docker daemon itself.

We supply both our containers with `--restart unless-stopped` so the daemon will launch them as soon as it starts (unless we manually stop them).  We just need to make sure the [docker daemon starts at boot](https://docs.docker.com/install/linux/linux-postinstall//#configure-docker-to-start-on-boot):  
```sh
sudo systemctl enable docker
sudo reboot

# Wait for it to reboot

ssh jetson-nano.local
# Try Rhasspy TTS to make sure Rhasspy is running:
curl -X POST -d "hello world" http://localhost:12101/api/text-to-speech
```

And `docker ps` should output something similar to:
```
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                      NAMES
e797909decb6        synesthesiam/rhasspy-server:latest    "/run.sh --user-prof…"   32 hours ago        Up About a minute   0.0.0.0:12101->12101/tcp   rhasspy
06cf50964e98        homeassistant/home-assistant:stable   "/bin/entry.sh pytho…"   33 hours ago        Up About a minute   0.0.0.0:8123->8123/tcp     hass
```

Be careful about using `docker ps` to verify everything is starting at boot.  If the daemon didn't start, `docker ps` will start it- along with the containers.

## docker-compose

With multiple containers and so many options things are starting to get hairy.  We can reign things in with [docker compose](https://docs.docker.com/compose/).  Create `docker-compose.yml` with mostly the same contents as the `docker run` commands:
```yml
version: '3'
services:
    hass:
        image: "homeassistant/home-assistant:stable"
        restart: unless-stopped
        volumes:
            - "$HOME/.homeassistant:/config"
        ports:
            - "8123:8123"
        environment:
            - TZ=Asia/Nicosia
    rhasspy:
        image: "synesthesiam/rhasspy-server:latest"
        restart: unless-stopped
        volumes:
            - "$HOME/.config/rhasspy/profiles:/profiles"
        ports:
            - "12101:12101"
        devices:
            - "/dev/snd:/dev/snd"
            - "/dev/bus/usb:/dev/bus/usb"
        command: --user-profiles /profiles --profile en
```

We don't need the `--network` option because the [desired networking is implicitly provided by compose](https://docs.docker.com/compose/networking/).

Install docker compose and launch our containers:  
```sh
# Install pre-requisites
sudo apt install -y python-pip libffi-dev libssl-dev
# Install docker-compose
sudo pip install docker-compose
# Start containers. `-d` for detached mode
docker-compose up -d
```

If you check Rhasspy, you may see the following problem:  
```
HomeAssistantIntentHandler	Can't contact server	Unable to reach your Home Assistant server at http://hass:8123. Is it running?
```

Rhasspy starts before Hass is ready and listening.  If you test further you'll find that it has already retried and connected.  This behavior aligns with the "docker ethos" of fault-tolerant containers.  Should it really bother you, feel free to investigate the rickety world of container dependencies:  
- https://docs.docker.com/compose/startup-order/ and [depends-on](https://docs.docker.com/compose/compose-file/#depends_on)
- SO [#1](https://stackoverflow.com/questions/31746182/docker-compose-wait-for-container-x-before-starting-y), [#2](https://stackoverflow.com/questions/43671482/how-to-run-docker-compose-up-d-at-system-start-up/)
