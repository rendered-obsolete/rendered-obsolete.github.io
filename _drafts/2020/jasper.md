---
layout: post
title: Home Automation with Jasper
tags:
- jasper
- iot
canonical_url:
---

[https://mycroft.ai/](Mycroft) ([github](https://github.com/MycroftAI))


https://devtalk.nvidia.com/default/topic/1052358/jetson-tx1/cannot-record-audio/
```sh
arecord -l
```
https://help.ubuntu.com/community/VNC/Servers
https://askubuntu.com/questions/1035598/ubuntu-18-04-lts-x11vnc-no-longer-works
```
**** List of CAPTURE Hardware Devices ****
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 0: ADMAIF1 CIF ADMAIF1-0 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 1: ADMAIF2 CIF ADMAIF2-1 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 2: ADMAIF3 CIF ADMAIF3-2 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 3: ADMAIF4 CIF ADMAIF4-3 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 4: ADMAIF5 CIF ADMAIF5-4 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 5: ADMAIF6 CIF ADMAIF6-5 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 6: ADMAIF7 CIF ADMAIF7-6 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 7: ADMAIF8 CIF ADMAIF8-7 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 8: ADMAIF9 CIF ADMAIF9-8 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: tegrasndt210ref [tegra-snd-t210ref-mobile-rt565x], device 9: ADMAIF10 CIF ADMAIF10-9 []
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: HD3000 [Microsoft® LifeCam HD-3000], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

arecord -t wav -c 2 -d 10 -v tmp_file.wav
aplay tmp_file.wav


https://devtalk.nvidia.com/default/topic/1049272/jetson-nano/jetson-nano-no-sound-output/

system settings -> sound -> select HDMI/DisplayPort

```sh
sudo apt-get install pulseaudio
pactl list short sources
```
Output:
```
0	alsa_output.platform-70030000.hda.hdmi-stereo.monitor	module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
1	alsa_output.platform-sound.analog-stereo.monitor	module-alsa-card.c	s16le 2ch 48000Hz	SUSPENDED
2	alsa_input.platform-sound.analog-stereo	module-alsa-card.c	s16le 2ch 44100Hz	SUSPENDED
5	alsa_input.usb-Microsoft_Microsoft___LifeCam_HD-3000-02.analog-mono	module-alsa-card.c	s16le 1ch 48000Hz	SUSPENDED
```



sudo vim /etc/pulse/default.pa
```sh
rm -rf ~/.config/pulse/
sudo shutdown -r now
```

Force display on
```sh
xset -display :0 dpms force on
```

https://elinux.org/Jetson/Cameras#Disabling_USB_auto-suspend
https://elinux.org/Jetson/Performance#How_to_run_a_command_with_root_privileges_temporarily_or_on_every_bootup
https://www.jetsonhacks.com/2015/05/27/usb-autosuspend-nvidia-jetson-tk1/

https://www.bose.com/en_us/support/article/adding-another-music-source-soundlink-revolve.html
https://community.bose.com/t5/Portable-Archive/Using-Soundlink-Revolve-as-computer-mic/td-p/127960

## Jasper

[Manual install](https://jasperproject.github.io/documentation/installation/#manual-installation)

```sh
sudo apt-get update
sudo apt-get install nano git-core python-dev bison libasound2-dev libportaudio-dev python-pyaudio --yes
git clone https://github.com/jasperproject/jasper-client.git jasper
sudo apt-get install python-pip
sudo pip install --upgrade setuptools
sudo pip install -r jasper/client/requirements.txt
./jasper/jasper.py
```

Fail with:
```
ERROR:root:Error occured!
Traceback (most recent call last):
  File "./jasper/jasper.py", line 146, in <module>
    app = Jasper()
  File "./jasper/jasper.py", line 79, in __init__
    with open(new_configfile, "r") as f:
IOError: [Errno 2] No such file or directory: '/home/jake/.jasper/profile.yml'
```

https://jasperproject.github.io/documentation/configuration/


```sh
# Sphinxbase/Pocketsphinx
sudo apt-get install pocketsphinx python-pocketsphinx
# CMUCLMTK
sudo apt-get install subversion autoconf libtool automake gfortran g++ --yes
svn co https://svn.code.sf.net/p/cmusphinx/code/trunk/cmuclmtk/
cd cmuclmtk/
./autogen.sh && make && sudo make install
cd ..
```

```
libtool: compile:  g++ -DHAVE_CONFIG_H -I./../include -g -O2 -MT fst.lo -MD -MP -MF .deps/fst.Tpo -c fst.cc  -fPIC -DPIC -o .libs/fst.o
In file included from ./../include/fst/accumulator.h:36:0,
                 from ./../include/fst/label-reachable.h:32,
                 from ./../include/fst/lookahead-matcher.h:28,
                 from ./../include/fst/matcher-fst.h:26,
                 from fst.cc:26:
./../include/fst/replace.h: In constructor 'fst::ArcIterator<fst::ReplaceFst<A, T> >::ArcIterator(const fst::ReplaceFst<A, T>&, fst::ArcIterator<fst::ReplaceFst<A, T> >::StateId)':
./../include/fst/replace.h:1069:46: error: expected ';' before '::' token
       (fst_.GetImpl())->template CacheImpl<A>::InitArcIterator(state_,
                                              ^~
Makefile:328: recipe for target 'fst.lo' failed
```

OpenFST [1.3.x doesn't build with gcc 7](https://github.com/hfst/hfst/issues/358).




Need to install gcc 4.8 ([link0](https://linuxize.com/post/how-to-install-gcc-compiler-on-ubuntu-18-04/) [link1](https://askubuntu.com/questions/923337/installing-an-older-gcc-version3-4-3-on-ubuntu-14-04-currently-4-8-installed)):

```sh
# Install gcc 4.8
sudo apt-get install -y gcc-4.8 g++-4.8
# Setup 4.8 as gcc alternative
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 48 --slave /usr/bin/g++ g++ /usr/bin/g++-4.8 --slave /usr/bin/gcov gcov /usr/bin/gcov-4.8
# Switch to gcc 4.8
sudo update-alternatives --config gcc

make clean
make -j2 && sudo make install
# Wait 35-40 minutes
```





## Newer OpenFST and Phoneetisaurus

http://www.openfst.org/twiki/bin/view/FST/FstDownload

```sh
tar xf openfst-1.6.3.tar.gz
./configure --enable-compact-fsts --enable-const-fsts --enable-far --enable-lookahead-fsts --enable-pdt
make -j2 # or -j4
sudo make install
```

https://github.com/AdolfVonKleist/Phonetisaurus
`./configure` fails:
```
configure: error: Can't find OpenFST or one or more of its extensions. Use --with-openfst-includes and --with-openfst-libs to specify where you have installed OpenFst. OpenFst should have been configured with the following flags: --enable-static --enable-shared --enable-far --enable-ngram-fsts
```

```
./configure --enable-compact-fsts --enable-const-fsts --enable-far --enable-lookahead-fsts --enable-pdt --enable-static --enable-shared --enable-ngram-fsts
make -j2 && sudo make install

pip install pybindgen
./configure --enable-python
make -j2
sudo make install
```

Problem is jasper doesn't work with newer Phonetisaurus.