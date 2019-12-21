---
layout: post
title: Home Assistant Voice Recognition with Rhasspy
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

With the [impending demise of Snips]({% post_url /2019/2019-12-10-respeaker %}), I've been looking for a suitable replacement offline speech recognition solution.  After some research, [Rhasspy](https://github.com/synesthesiam/rhasspy) seems like a real winner.  Besides supporting a variety of toolkits, it has [good documentation](https://rhasspy.readthedocs.io/en/latest/), and is easy to get working.

Here I'm runnning both Hass and Rhasspy on the same Raspberry Pi.  From my PC I connect to the pi as `pi3.local`- adjust based on the name of your device or use the IP address.  If working directly on the device everything is `localhost`.

## Installation

Normally, I like to go through manual installation.  But installing Pocketsphinx and OpenFST for [Jasper](http://jasperproject.github.io/documentation/) was enough of a headache that I decided to go the container route.

Follow the [Rhasspy installation docs](https://rhasspy.readthedocs.io/en/latest/installation/):

If you haven't already, [install docker](https://docs.docker.com/install/linux/docker-ce/debian/) using the [convenience script](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-convenience-script):  
```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

Run Rasspy with the recommended:  
```sh
docker run -d -p 12101:12101 \
      --restart unless-stopped \
      -v "$HOME/.config/rhasspy/profiles:/profiles" \
      --device /dev/snd:/dev/snd \
      synesthesiam/rhasspy-server:latest \
      --user-profiles /profiles \
      --profile en
```

__Or__, use docker-compose:

1. [Install docker-compose](https://docs.docker.com/compose/) via [alternative install options](https://docs.docker.com/compose/install/#alternative-install-options):
    ```sh
    sudo pip install docker-compose
    ```
1. Place the following in docker-compose.yml:
    ```yaml
    rhasspy:
        image: "synesthesiam/rhasspy-server:latest"
        restart: unless-stopped
        volumes:
            - "$HOME/.config/rhasspy/profiles:/profiles"
        ports:
            - "12101:12101"
        devices:
            - "/dev/snd:/dev/snd"
        command: --user-profiles /profiles --profile en
    ```
1. Run: `docker-compose up`

If `docker-compose up` fails with `ImportError: No module named ssl_match_hostname` see [this issue](https://github.com/docker/docker-py/issues/1502):
```sh
# Remove problematic `ssl-match-hostname`
sudo pip uninstall backports.ssl-match-hostname docker-compose
# Install alternative `ssl-match-hostname`
sudo apt-get install -y python-backports.ssl-match-hostname \
    python-backports.shutil-get-terminal-size
# Reinstall docker-compose
sudo pip install docker-compose
```

## Configuration

Once docker outputs `rhasspy_1  | Running on https://0.0.0.0:12101 (CTRL + C to quit)` it should be up and running.  Ignore what it says and use `http` instead of `https`- point your browser at http://pi3.local:12101.

The first things to look at are audio [input](https://rhasspy.readthedocs.io/en/latest/audio-input/) and [output](https://rhasspy.readthedocs.io/en/latest/audio-output/).

I was able to configure everything via the __Settings__ tab.  Should that not cooperate, everything can be done [via json](https://rhasspy.readthedocs.io/en/latest/profiles/#available-settings).  In `$home/.config/rhasspy/profiles/default.json`:
```json
"microphone": {
  "system": "pyaudio",
  "pyaudio": {
    "device": "4",
    "frames_per_buffer": 480
  }
}
```



For more insight into what PyAudio sees (from [SO#1](https://stackoverflow.com/questions/20760589/list-all-audio-devices-with-pythons-pyaudio-portaudio-binding), [SO#2](https://stackoverflow.com/questions/36894315/how-to-select-a-specific-input-device-with-pyaudio)):
```sh
# Install pyaudio
sudo apt-get install -y python-pyaudio
# Launch python REPL
python
```
Run the following:
```py
import pyaudio
p = pyaudio.PyAudio()
for i in range(p.get_device_count()):
    print p.get_device_info_by_index(i)
```
Or
```py
import pyaudio
p = pyaudio.PyAudio()
info = p.get_host_api_info_by_index(0)
numdevices = info.get('deviceCount')
for i in range(0, numdevices):
        if (p.get_device_info_by_host_api_device_index(0, i).get('maxInputChannels')) > 0:
            print "Input Device id ", i, " - ", p.get_device_info_by_host_api_device_index(0, i).get('name')
```




Show running containers with `docker ps` or `docker container ls`:
```sh
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                      NAMES
4181a2880c84        synesthesiam/rhasspy-server:latest   "/run.sh --user-prof…"   26 hours ago        Up 4 minutes        0.0.0.0:12101->12101/tcp   pi_rhasspy_1
```
Get a shell to the container:
```sh
docker exec -it pi_rhasspy_1 /bin/bash
root@4181a2880c84:/#
```

Here you can use `arecord` and `aplay` to test.

From the __Speech__ tab, use __Hold to Record__ or __Tap to Record__ for speech input.  Saying "what time is it" should output:
```json
{
    "intent":{
        "entities":{},
        "hass_event":{
            "event_data":{},
            "event_type": "rhasspy_GetTime"
        },
        "intent":{
            "confidence": 1,
            "name": "GetTime"
        },
        "raw_text": "what time is it",
    }
}
```

Test TTS:
1. __Speech__ tab, enter text for __Sentence__ and hit __Speak__.
1. Check __Log__ tab for command lines it's using
1. __Settings__ tab __Text to Speech__ and select one of the other TTS engines (eSpeak didn't work for me, but flite and pico-tts did)

## Home Assistant Integration

Uses the [Hermes protocol over MQTT](https://rhasspy.readthedocs.io/en/latest/usage/#mqtt)- just like Snips!

Hass' REST API and [POSTing to `/api/events`](https://rhasspy.readthedocs.io/en/latest/usage/#home-assistant) endpoint.

https://www.home-assistant.io/docs/authentication/


1. Hass: Create [long-lived access token](https://developers.home-assistant.io/docs/en/auth_api.html#long-lived-access-token)
    1. http://pi3.local:8123/profile under __Long-Lived Access Tokens > Create Token__
1. Rhasspy: Configure intent handling with Hass
    1. Open http://pi3.local:12101/
    1. __Settings > Intent Handling__
    1. __Hass URL__ `http://172.17.0.1:8123` (docker host, `172.17.0.2` is the container itself)
        - If not using docker could instead use `localhost`
    1. __Access Token__ the token from above
    1. __Save Settings > OK__ to restart


[Hass REST API](https://developers.home-assistant.io/docs/en/external_api_rest.html) is working:
```sh
curl -X GET -H "Authorization: Bearer <ACCESS TOKEN>" -H "Content-Type: application/json" http://pi3.local:8123/api/
```
```json
{"message": "API running."}
```

```
docker exec -it pi_rhasspy_1 /bin/bash
curl -X GET -H "Authorization: Bearer <ACCESS TOKEN>" -H "Content-Type: application/json" http://localhost:8123/api/
curl: (7) Failed to connect to localhost port 8123: Connection refused
```



__Developer Tools > Events > Listen to events__ use `rhasspy_GetTime` and __Start Listening__.

Say "what time is it" and Hass should output:
```json
{
    "event_type": "rhasspy_GetTime",
    "data": {},
    "origin": "REMOTE",
    "time_fired": "2019-12-17T16:02:51.366090+00:00",
    "context": {
        "id": "012345678901234567890123456789",
        "parent_id": null,
        "user_id": "deadbeefdeadbeefdeadbeefdeadbeef"
    }
}
```

__Configuration > Automation > +__
[Event trigger](https://www.home-assistant.io/docs/automation/trigger/#event-trigger)

- Triggers
    - Trigger type: __Event__
- Actions
    - Action type: __Call service__
    - Service: `system_log.write`
    - Service data: `{message: 'Hello event'}`

__Developer Tools > Logs__ should show the message.

## Text-To-Speech

To use Rhasspy's TTS we can leverage its [REST API](https://rhasspy.readthedocs.io/en/latest/reference/#http-api):
```sh
curl -X POST -d "hello world" http://pi3.local:12101/api/text-to-speech
```

https://www.home-assistant.io/integrations/rest_command/.  In `configuration.yaml`:
```yaml
rest_command:
  tts:
    url: http://localhost:12101/api/text-to-speech
    method: POST
    payload: '{{ message }}'
```

The `payload` is Jinja2 template that can be set by the caller.

__Developer Tools > Services__ `rest_command.tts` service and with data `message: "hello"` and __Call Service__.

__Configuration > Automation__ and edit the previous thing __Add Action__:
- Action type: __Call service__
- Service: `rest_command.tts` (it should auto-complete for you)
- Service data: `{message: 'hello world'}`

"What time is it" should trigger a full loop:
```
speech -> Rhasspy -> intent -> Hass -> text -> Rhasspy -> speech
```

## Systemd

I'd like Rhasspy to [auto-start similar to Hass]({% post_url /2019/2019-11-22-home_assistant %}#auto-start).

It would seem that mixing docker with systemd is bad mojo, making me contemplate re-[installing Hass via docker](https://www.home-assistant.io/docs/installation/docker/).  Docker says little on starting containers with systemd other than [don't cross the streams with restarts](https://docs.docker.com/config/containers/start-containers-automatically/#use-a-process-manager).  And so far google has turned up dubious results- mostly from several years ago that don't work with current versions of docker.

Create `/etc/systemd/system/rhasspy@homeassistant.service`:
```ini
[Unit]
Description=Rhasspy
Wants=home-assistant@homeassistant.service 
Requires=docker.service
After=home-assistant@homeassistant.service docker.service

[Service]
Type=exec
ExecStart=docker run --rm \
	--name rhasspy \
	-p 12101:12101 \
	-v "/home/homeassistant/.config/rhasspy/profiles:/profiles" \
	--device /dev/snd:/dev/snd \
	synesthesiam/rhasspy-server:latest \
	--user-profiles /profiles --profile en 
ExecStop=docker stop rhasspy
# Restart on failure
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

| | | |
|-|-|-|
|Wants/Requires/After| Docker __must__ be running, and ideally Hass is (but we can start Rhasspy without it) | [man](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#%5BUnit%5D%20Section%20Options)
|Type| Stronger requirement than `simple` ensuring the process starts | [man](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=)
|ExecStart|Start the container
|ExecStop|Stop the container

Differences from the example `docker run`:
| | | |
|-|-|-|
| `--rm` | Remove the container on exit.  Otherwise we get "name taken" errors on restarts.
| `--name` | Give it a predictable name to simplify `ExecStop`, and make it easier to work with
| `-v` | Docker defaults to creating files as root.  `/srv/` might be better, but I thought this would make the profiles easier to find.
| `--restart unless-stopped`| Removed since systemd is managing the lifetimee.


```sh
sudo systemctl --system daemon-reload
sudo systemctl enable rhasspy@homeassistant
sudo systemctl start rhasspy@homeassistant
```

`docker container ls` open 
Check output of `sudo journalctl -f -u rhasspy@homeassistant`.


If you fail to remove `$HOME` it will fail with:  
```
Dec 18 19:00:25 pi3 docker[4764]: /usr/bin/docker: invalid reference format.
```