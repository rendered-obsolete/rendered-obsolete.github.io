---
layout: post
title: Home Assistant Offline Voice Recognition with Snips
tags:
- smarthome
- iot
- homeautomation
- hass
series: Smart Home
---

After the [intial install of __Home Assistant__]({% post_url /2019/2019-11-22-home_assistant %}), I've beenn eager to get some basic voice recognition working.  One of my early goals was for it to be "offline"; meaning, not use Amazon or Google.

## Hardware

- [Raspberry Pi 3 model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
    - Running [Raspbian Stretch]({% post_url /2019/2019-03-21-raspi_3 %})
- A USB microphone
    - [ReSpeaker USB 4 Mic Array](https://respeaker.io/usb_4_mic_array/)
    - [Microsoft LifeCam HD-3000](https://www.microsoft.com/accessories/en-us/products/webcams/lifecam-hd-3000/t3h-00011)

I was originally working with the HD-3000, but wasn't very happy with the recording quality.  I'm still experimenting with the ReSpeaker, but it definitely seems better.  In any case, configuration was pretty similar- and likely the same goes for any other USB microphone.

## Basic Alsa Audio

First, we need to get audio working; both a microphone and speaker.

Good, concise documentation that explains what's going on with Raspberry Pi/Debian audio has eluded me thus far.  Most of this is extracted from random forum posts, Stack Overflow, and a smattering of trial and error.

You can record from a microphone with `arecord`.  Abridged `arecord --help` output:
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
  <SNIP>
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

To record using the ReSpeaker (card `ArrayUAC10`):
```sh
# `-d 3` records for 3 seconds (otherwise `Ctrl+c` to stop)
# `-D` sets the PCM device
arecord -d 3 -D hw:ArrayUAC10 tmp_file.wav
```

It may output:
```
Recording WAVE 'tmp_file.wav' : Unsigned 8 bit, Rate 8000 Hz, Mono
arecord: set_params:1299: Sample format non available
Available formats:
- S16_LE
```

Like `arecord -L` says, `hw:` is "Direct hardware device without any conversions".  We either need to record in a supported format, or use `plughw:` ("Hardware device with all software conversions").  Either of these work:
```sh
arecord -d 3 -D plughw:ArrayUAC10 tmp_file.wav
# `-f S16_LE` signed 16-bit little endian
# `-c 6` six channels
# `-r 16000` 16kHz
arecord -f S16_LE -c 6 -r 16000 -d 3 -D hw:ArrayUAC10 tmp_file.wav
```

You can get a list of supported parameters with `arecord --dump-hw-params -D hw:ArrayUAC10`:
```
HW Params of device "hw:ArrayUAC10":
--------------------
ACCESS:  MMAP_INTERLEAVED RW_INTERLEAVED
FORMAT:  S16_LE
SUBFORMAT:  STD
SAMPLE_BITS: 16
FRAME_BITS: 96
CHANNELS: 6
RATE: 16000
PERIOD_TIME: [1000 2730625]
PERIOD_SIZE: [16 43690]
PERIOD_BYTES: [192 524280]
PERIODS: [2 1024]
BUFFER_TIME: [2000 5461313)
BUFFER_SIZE: [32 87381]
BUFFER_BYTES: [384 1048572]
TICK_TIME: ALL
--------------------
```

In online resources you'll see values similar to `hw:2,0`, which means "card 2, device 0".  Looking at the `arecord -l` output, it's the same as `hw:ArrayUAC10` since the ReSpeaker only has the one device.

You can play the recorded audio with `aplay`.  Looking at the output from `aplay -L`, I can:
```sh
aplay -D plughw:SoundLink tmp_file.wav
```

There's at least two configuration files that can affect behaviour of `arecord`/`aplay`:
- `/etc/asound.conf`
- `~/.asoundrc`

For example, after changing the default sound card via __Audio Device Settings__ my `~/.asoundrc` contains:
```
pcm.!default {
	type hw
	card 2
}

ctl.!default {
	type hw
	card 2
}
```

If I check `aplay -l`, "card 2" is my _Bose Revolve SoundLink_ USB speaker:
```
**** List of PLAYBACK Hardware Devices ****
card 0: ALSA [bcm2835 ALSA], device 0: bcm2835 ALSA [bcm2835 ALSA]
  <SNIP>
card 0: ALSA [bcm2835 ALSA], device 1: bcm2835 IEC958/HDMI [bcm2835 IEC958/HDMI]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: ALSA [bcm2835 ALSA], device 2: bcm2835 IEC958/HDMI1 [bcm2835 IEC958/HDMI1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: Dummy [Dummy], device 0: Dummy PCM [Dummy PCM]
  <SNIP>
card 2: SoundLink [Bose Revolve SoundLink], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

## Voice Recognition with Snips

At first I was thrilled to find [Snips](https://snips.ai/):

- Completely offline (once you create and download the assistant)
- [Good integration with Home Assistant](https://www.home-assistant.io/integrations/snips)
- Simple to install and configure

But after initial (successful) experimentation, there's a few largish problems:

- [May be issues on Debian Buster](https://github.com/snipsco/snips-issues/issues/161)
- [Dubious support for devices other than Raspberry Pi](https://docs.snips.ai/articles/other-platforms)
- Worst of all, post-acquisition [they're killing the "console" web-app you need to make assistants](https://forum.snips.ai/t/important-message-regarding-the-snips-console/4145)

Oops.  Hopefully it will return in another form.

### Install

Following the [manual setup instructions](https://docs.snips.ai/articles/raspberrypi/manual-setup):
```sh
sudo apt-get install -y dirmngr
sudo bash -c  'echo "deb https://raspbian.snips.ai/$(lsb_release -cs) stable main" > /etc/apt/sources.list.d/snips.list'
sudo apt-key adv --fetch-keys https://raspbian.snips.ai/531DD1A7B702B14D.pub
sudo apt-get update
sudo apt-get install -y snips-platform-voice
```

### Create an Assistant

An "assistant" defines what voice commands Snips handles.  Need to create an assistant via the (soon to be shutdown) [Snips Console](https://console.snips.ai/):

1. Click __Add an App__
1. Click `+` of apps of interest
1. Click __Add Apps__ button
1. Wait for training to complete
1. Click __Deploy Assistant__ button
1. __Download and install manually__

[Install the assistant](https://docs.snips.ai/articles/console/actions/deploy-your-assistant#deploy-your-assistant-manually-without-sam):
```sh
pc> scp ~/Downloads/assistant_proj_XYZ.zip pi@pi3.local:~
pc> ssh pi@pi3.local

sudo rm -rf /usr/share/snips/assistant/
sudo unzip ~/assistant_proj_1mE9N2ylKWa.zip -d /usr/share/snips/
sudo systemctl restart 'snips-*'
```

At this point Snips should be working.  If triggered with the wake word (default is `hey snips`), it should send "intents" over [MQTT].

### Verification/Troubleshooting

Check all services are green and `active (running)`:
```sh
sudo systemctl status 'snips-*'
```

Initially, the __Snips Audio Server__ was unable to start.  Check output in syslog:
```sh
tail -f /var/log/syslog
```

It was unable to open the "default" audio capture device:
```
Dec  5 07:22:25 pi3 snips-audio-server[28216]: INFO:snips_audio_alsa::capture: Starting ALSA capture on device "default"
Dec  5 07:22:25 pi3 snips-audio-server[28216]: ERROR:snips_audio_server       : an error occured in the audio pipeline: Error("snd_pcm_open", Sys(ENOENT))
Dec  5 07:22:25 pi3 snips-audio-server[28216]:  -> caused by: ALSA function 'snd_pcm_open' failed with error 'ENOENT: No such file or directory'
```

We could set the "default" device.  Or, `/etc/snips.toml` contains [platform configuration](https://docs.snips.ai/articles/platform/platform-configuration) where we can specify values from [above](#basic-alsa-audio):
```toml
[snips-audio-server]
alsa_capture = "plughw:ArrayUAC10"
alsa_playback = "plughw:SoundLink"
```

[snips-watch](https://docs.snips.ai/articles/platform/snips-watch) shows a lot of information:
```sh
sudo apt-get install -y snips-watch
snips-watch -vv
```

I installed the [weather app](https://console.snips.ai/store/en/skill_lbVbZQn5Mnk).  So, if I say, "hey snips, what's the weather?" snips-watch should output:
```
[15:00:52] [Hotword] detected on site default, for model hey_snips
[15:00:52] [Asr] was asked to stop listening on site default
[15:00:52] [Hotword] was asked to toggle itself 'off' on site default
[15:00:52] [Dialogue] session with id 'e39a4367-e167-467c-912a-e047f49bea7a' was started on site default
[15:00:52] [Asr] was asked to listen on site default
[15:00:54] [Asr] captured text "what 's the weather" in 2.0s with tokens: what[0.950], 's[0.950], the[1.000], weather[1.000]
[15:00:54] [Asr] was asked to stop listening on site default
[15:00:55] [Nlu] was asked to parse input "what 's the weather"
[15:00:55] [Nlu] detected intent searchWeatherForecast with confidence score 1.000 for input "what 's the weather"
[15:00:55] [Dialogue] New intent detected searchWeatherForecast with confidence 1.000
```

Instead of snips-watch, you can probably use any MQTT client:
```sh
sudo apt-get install -y mosquitto-clients
# Subscribe to all topics
mosquitto_sub -p 1883 -t "#"
```

## Home Assistant and Snips

Both Home Assistant and Snips are designed to use [MQTT].  You can either:  
- [Have Hass use Snips' broker](https://www.home-assistant.io/integrations/snips#specifying-the-mqtt-broker)
    - The Hass documentation incorrectly says the Snips broker is running on port 9898.  Currently the default is 1883, but consult `/etc/snips.toml`.
- [Have Snips use Hass' broker](https://docs.snips.ai/articles/platform/platform-configuration#common-parameters-snips-common)

Since we did everything from scratch, [Hass doesn't have a broker](https://www.home-assistant.io/docs/mqtt/broker/).  So, we can just point Hass at the one that got installed with Snips.  In `configuration.yml`:
```yml
# Enable snips (VERY IMPORTANT)
snips:
# Setup MQTT
mqtt:
  broker: 127.0.0.1
  port: 1883
```

Restart Hass and from the UI pick __☰ > Developer Tools > MQTT > Listen to a Topic__ and enter `hermes/intent/#` (all Snips intents) then __Start Listening__.

Now say "hey snips, what's the weather" and you should see a message for `searchWeatherForecast` intent pop up.

To test TTS, in __Developer Tools > Services__ try `snips.say` service with data `text: hello` and __Call Service__.  You should be greeted by a robo-voice from the speaker.

Let's try a basic [intent script](https://www.home-assistant.io/integrations/intent_script/) triggered on the intent.  In `configuration.yaml`:
```yaml
intent_script:
  searchWeatherForecast:
    speech:
      text: 'hello intent'
    action:
      - service: system_log.write
        data_template:
          message: 'Hello intent'
          level: warning
```

Now when hass receives the intent, the TTS engine will say "hello intent" and output something to __Developer > Logs__.

## The End?

It's a total bummer the future of Snips is uncertain because it was perfect for voice controlled home automatic.  But, that would be why it was acquired.


[mqtt]: http://mqtt.org/