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

With the [impending demise of Snips]({% post_url /2019/2019-12-10-respeaker %}#voice-recognition-with-snips), I've been looking for a suitable replacement offline speech recognition solution.  After some research, [Rhasspy](https://github.com/synesthesiam/rhasspy) seems like a real winner.  Besides supporting a variety of toolkits, it has [good documentation](https://rhasspy.readthedocs.io/en/latest/), and can be easy to get working.

A [Raspberry Pi3B with stock Debian]({% post_url /2019/2019-03-21-raspi_3 %}) really struggled with this at times.  It might be possible to alleviate this by picking different services or adjusting other configuration, but you might be better off just using a more powerful device (like a [Pi4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) or [Jetson Nano]({% post_url /2019/2019-04-01-jetson_nano %})) or [running it remotely](https://rhasspy.readthedocs.io/en/latest/speech-to-text/#remote-http-server).


## Installation

Normally, I like to go through manual installation.  But installing Pocketsphinx and OpenFST for [Jasper](http://jasperproject.github.io/documentation/) was enough of a headache that I decided to go the container route.

Follow the [Rhasspy installation docs](https://rhasspy.readthedocs.io/en/latest/installation/).  I'm runnning both Hass and Rhasspy on the same Raspberry Pi.  From my PC I connect to the pi as `pi3.local`- adjust this based on the name of your device or use the IP address.  If working directly on the device everything is `localhost`.

If you haven't already, [install docker](https://docs.docker.com/install/linux/docker-ce/debian/) using the [convenience script](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-convenience-script):  
```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

You can run Rasspy with the recommended:  
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
1. Use the recommended `docker-compose.yml`:
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

## Docker Shell

When running things with docker, it takes an extra step to have a shell in the context of the container.

1. Show running containers with `docker ps` or `docker container ls`:
    ```sh
    CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                      NAMES
    4181a2880c84        synesthesiam/rhasspy-server:latest   "/run.sh --user-prof…"   26 hours ago        Up 4 minutes        0.0.0.0:12101->12101/tcp   pi_rhasspy_1
    ```
1. Get a shell to the container:
    ```sh
    docker exec -it pi_rhasspy_1 /bin/bash
    # Now you're in the container
    root@4181a2880c84:/#
    ```

Replace `pi_rhasspy_1` with the "container id" or "name" of the appropriate container.

## Configuration

Once docker outputs `rhasspy_1  | Running on https://0.0.0.0:12101 (CTRL + C to quit)` Rhasspy should be up and running.  Ignore what it says and use `http` instead of `https`- point your browser at [http://pi3.local:12101].

At this point I was able to configure everything via the __Settings__ tab.  Should that not cooperate, everything can also be done [via json](https://rhasspy.readthedocs.io/en/latest/profiles/#available-settings).

### Audio

The first things to get working are audio [input](https://rhasspy.readthedocs.io/en/latest/audio-input/) and [output](https://rhasspy.readthedocs.io/en/latest/audio-output/).  Refer back to an [earlier post about working with ALSA]({% post_url /2019/2019-12-10-respeaker %}#basic-alsa-audio).

1. __Settings > Microphone__ ("Audio Recording")
    1. __Use arecord directly (ALSA)__ (default is PyAudio)
    1. Select appropriate __Input Device__
1. __Settings > Sounds__ ("Audio Playing")
    1. __Use aplay directly (ALSA)__
    1. Select appropriate __Output Device__

To verfy audio recording/playback works, from a [docker shell](#docker-shell) use `arecord` and `aplay`.

If, instead of ALSA for input you want to use PyAudio, it's handy to see what PyAudio sees:
```sh
# Install pyaudio
sudo apt-get install -y python-pyaudio
# Launch python REPL
python
```

Then, run the following (from [SO#1](https://stackoverflow.com/questions/20760589/list-all-audio-devices-with-pythons-pyaudio-portaudio-binding), [SO#2](https://stackoverflow.com/questions/36894315/how-to-select-a-specific-input-device-with-pyaudio)):
```py
import pyaudio
p = pyaudio.PyAudio()
for i in range(p.get_device_count()):
    print p.get_device_info_by_index(i)

## OR

import pyaudio
p = pyaudio.PyAudio()
info = p.get_host_api_info_by_index(0)
numdevices = info.get('deviceCount')
for i in range(0, numdevices):
        if (p.get_device_info_by_host_api_device_index(0, i).get('maxInputChannels')) > 0:
            print "Input Device id ", i, " - ", p.get_device_info_by_host_api_device_index(0, i).get('name')
```

### Rhasspy TTS

Testing text-to-speech also seems to be the easist way to validate your audio output is working.

1. __Settings > Text to Speech__
    - `eSpeak` didn't work for me, but both `flite` and `pico-tts` did
1. __Speech__ tab, in __Sentence__ put `hello` and __Speak__
1. Check __Log__ tab for `FliteSentenceSpeaker` lines to see e.g. command lines it's using

### Intent Recognition

One way to validate audio input is to setup Rhasspy to recognize intents.

1. __Settings > Intent Recognition__
    - Default `OpenFST` should work
1. __Sentences__ tab to configure recognized intents
    - Uses a [simplified JSGF syntax](https://rhasspy.readthedocs.io/en/latest/training/#sentencesini)
1. __Speech__ tab, use __Hold to Record__ or __Tap to Record__ for mic input
1. Saying `what time is it` should output:
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

### Wake word

Another way to validate audio input is to setup a phrase to trigger Rhasspy to recognize intents (i.e. `hey siri`, `ok google`, etc.)

1. __Settings > Wake Word__
1. __PocketSphinx__ is the only fully open/offline option
    - "Wake Keyphrase" is the trigger phrase
1. __Save Settings__ and wait for Rhasspy to restart
1. __Train__ (mentioned in the [docs](https://rhasspy.readthedocs.io/en/latest/wake-word/#pocketsphinx))
1. Check __Log__ for `PocketsphinxWakeListener: Hotword detected`

If your wake keyphrase contains a new word, the log will complain it's not in `dictionary.txt` after you __Save Settings__:
```
[WARNING:955754080] PocketsphinxWakeListener: XXX not in dictionary
[DEBUG:3450672] PocketsphinxWakeListener: Loading wake decoder with hmm=/profiles/en/acoustic_model, dict=/profiles/en/dictionary.txt
```

It seems like either adding a custom word via the __Words__ tab and/or hitting __Train__ should fix this, but I haven't yet figured out the correct incantation.

## Hass Integration

Integrating with Home Assistant is accomplished by leveraging Hass' REST API and [POSTing to `/api/events`](https://rhasspy.readthedocs.io/en/latest/usage/#home-assistant) endpoint.


1. Hass: Create [long-lived access token](https://developers.home-assistant.io/docs/en/auth_api.html#long-lived-access-token)
    1. Open Hass user profile: [http://pi3.local:8123/profile] 
    1. __Long-Lived Access Tokens > Create Token__
    - Also read [Hass authetication docs](https://www.home-assistant.io/docs/authentication/)
1. Rhasspy: Configure intent handling with Hass
    1. Open Rhasspy: [http://pi3.local:12101]
    1. __Settings > Intent Handling__
    1. __Hass URL__ `http://172.17.0.1:8123` (the docker host, `172.17.0.2` is the container itself)
        - If not using docker could instead use `localhost`
    1. __Access Token__ the token from above
    1. __Save Settings > OK__ to restart


Check [Hass REST API](https://developers.home-assistant.io/docs/en/external_api_rest.html) is working:
```sh
# Replace `<TOKEN>` with Hass Long-lived access token
curl -X GET -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" http://pi3.local:8123/api/
```
Should return:
```json
{"message": "API running."}
```

Note that from _within_ the container you can't connect to services _outside_ the container using `localhost`.  There's a few different ways to do this, but that's why we're using `172.17.0.1` above:
```sh
# Shell into container
docker exec -it pi_rhasspy_1 /bin/bash
# Try Hass REST API to `localhost`
# Replace `<TOKEN>` with Hass Long-lived access token
curl -X GET -H "Authorization: Bearer <TOKEN>" -H "Content-Type: application/json" http://localhost:8123/api/
curl: (7) Failed to connect to localhost port 8123: Connection refused
```

Let's test the Rhasspy->Hass connection:  

1. Open Hass: [http://pi3.local:8123]
1. __Developer Tools > Events > Listen to events__
    - `rhasspy_GetTime` and __Start Listening__.
1. Like for [intent recognition](#intent-recognition), say "what time is it"
1. Hass should output:
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

Let's test Hass automation:  

1. Open Hass: [http://pi3.local:8123]
1. __Configuration > Automation > +__
1. Create an [Event trigger](https://www.home-assistant.io/docs/automation/trigger/#event-trigger):
    - Triggers
        - Trigger type: __Event__
    - Actions
        - Action type: __Call service__
        - Service: `system_log.write`
        - Service data: `{message: 'Hello event'}`
1. Like for [intent recognition](#intent-recognition), say "what time is it"
1. In Hass, __Developer Tools > Logs__ should show the message.

## Hass TTS

To use Rhasspy's TTS we can leverage its [REST API](https://rhasspy.readthedocs.io/en/latest/reference/#http-api):
```sh
curl -X POST -d "hello world" http://pi3.local:12101/api/text-to-speech
```

To trigger this from Hass, we can use the [RESTful Command](https://www.home-assistant.io/integrations/rest_command/) integration.  In `configuration.yaml`:
```yaml
rest_command:
  tts:
    url: http://localhost:12101/api/text-to-speech
    method: POST
    payload: '{{ message }}'
```

The `payload` is Jinja2 template that can be set by the caller.

Test the `tts` REST command:

1. Open Hass: [http://pi3.local:8123]
1. __Developer Tools > Services__
1. Specify `rest_command.tts` service and with data `message: "hello"`
1. __Call Service__ to trigger Rhasspy TTS

Let's add it to our Hass automation:  

1. __Configuration > Automation__
1. Edit the previous item (click the pencil- ✎)
1. __Add Action__:
    - Action type: __Call service__
    - Service: `rest_command.tts` (it should auto-complete for you)
    - Service data: `{message: 'hello world'}`
1. Like for [intent recognition](#intent-recognition), say "what time is it"

This should trigger a full loop:
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
|ExecStop|Stop the container by name

For `ExecStart`, note a few differences from the [original `docker run`](#installation):  

| | | |
|-|-|-|
| `--rm` | Remove the container on exit.  Otherwise we get "name taken" errors on restarts.
| `--name` | Give it a predictable name to simplify `ExecStop`, and make it easier to open [docker shells](#docker-shell)
| `-v` | Docker defaults to creating files as root.  `/srv/` might be better, but I thought this would make the profiles easier to find.
| `--restart unless-stopped`| Removed since systemd is managing the lifetime.

Configure it to auto-start and start it:
```sh
sudo systemctl --system daemon-reload
sudo systemctl enable rhasspy@homeassistant
sudo systemctl start rhasspy@homeassistant
```

To debug:
```sh
# Check running containers
docker container ls
# Check log output
sudo journalctl -f -u rhasspy@homeassistant
# Open docker shell
docker exec -it rhasspy /bin/bash
```

Note, if you fail to remove `$HOME` from `docker run` it will fail with:  
```
Dec 18 19:00:25 pi3 docker[4764]: /usr/bin/docker: invalid reference format.
```


[http://pi3.local:12101]: http://pi3.local:12101
[http://pi3.local:8123]: http://pi3.local:8123
[http://pi3.local:8123/profile]: http://pi3.local:8123/profile