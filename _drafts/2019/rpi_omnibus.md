---
layout: post
title: Raspberry Pi 3 Raspbian Omnibus
tags:
- raspi
- linux
---

That day has come.

I've had a Raspberry Pi 3 with a touchscreen [running Windows 10 IoT Core]({% post_url /2018/2018-09-02-win10-iot-core-raspi %}) on my desk at work for a while now.  IoT Core isn't dead, I see it reboot for updates every now and then.

https://github.com/Microsoft?utf8=%E2%9C%93&q=iotcore&type=&language=

[Raspbian](https://www.raspberrypi.org/downloads/).

## 


__Rotate Screen 180 Degrees__  

In `/boot/config.txt` add:
```
lcd_rotate=2
```

__Screen Brightness__  

In the evening the default brightness is alarming:
```
sudo sh -c "echo 80 > /sys/class/backlight/rpi_backlight/brightness"
```

Max brightness seems to be `200`.  Up to `255` seems dimmer and higher than that results in "I/O Error".


__Virtual Keyboard__  

Regardless if you call it a "soft keyboard", "on-screen keyboard", or something else, you definitely want one.  Regardless if you prefer iOS or Android, prepare to be disappointed:
```
sudo apt-get install matchbox-keyboard
```

__SSH__
```
sudo raspi-config
```
_Interfacing Options > SSH > Yes_

When it comes to Linux configuration, one of the best investments you can make   is setting up "SSH Key-Based Authentication".  If that means nothing to you look into it, it will change your life.  There's numerous excellent guides on the internet, so I won't bother.

Speaking of investments, given the premium placed on screen real-estate don't overlook the `-X` option to ssh (on macOS/OSX you also need [XQuartz](https://www.xquartz.org/)):  

![](/assets/ssh_x_pi.png)

xrdp vs VNC

```
sudo raspi-config
```
_Interfacing Options > VNC > Yes_

## Visual Studio Code

[This post](https://www.hanselman.com/blog/BuildingVisualStudioCodeOnARaspberryPi3.aspx) seems to be the standard source for building VS Code.

```
git clone https://github.com/microsoft/vscode
cd vscode
./scripts/npm.sh install --arch=armhf
```

Launch VS Code:
```
./scripts/code.sh
```

You'll run into problems with `yarn` if you blindly attempt `apt-get install yarn` you end up `cmdtest`.  Following the [instructions from yarnpkg.com](https://yarnpkg.com/en/docs/install#debian-stable):
```bash
# Remove it "just in case"
sudo apt-get remove cmdtest
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
```

Install yarn and other pre-requisites:
```bash
sudo apt-get update
sudo apt-get install -y yarn libx11-dev libsecret-1-dev libxkbfile-dev
```

The last three are mentioned in the VS Code pre-requisites and without which your will end up with errors:
```

```


Likewise, if you `apt-get install` nodejs the version you get is too old and you'll run into errors like:
```
npm ERR! Error: Method Not Allowed
npm ERR!     at errorResponse (/usr/share/npm/lib/cache/add-named.js:260:10)
npm ERR!     at /usr/share/npm/lib/cache/add-named.js:203:12
npm ERR!     at saved (/usr/share/npm/node_modules/npm-registry-client/lib/get.js:167:7)
npm ERR!     at FSReqWrap.oncomplete (fs.js:135:15)
```

`./scripts/code.sh`:
```
[08:24:09] 'electron' errored after 11 s
[08:24:09] Error: No asset for version 3.1.3, platform linux and arch arm found
    at /home/pi/vscode/node_modules/gulp-atom-electron/src/download.js:83:15
    <SNIP CALLSTACK>
    github-releases-ms/node_modules/request/request.js:1083:12)
[08:24:17] Syncronizing built-in extensions...
[08:24:17] You can manage built-in extensions with the --builtin flag
[08:24:17] [marketplace] ms-vscode.node-debug@1.32.1 ✔︎
[08:24:17] [marketplace] ms-vscode.node-debug2@1.31.6 ✔︎
[08:24:17] [marketplace] ms-vscode.references-view@0.0.26 ✔︎
./scripts/code.sh: line 50: /home/pi/vscode/.build/electron/code-oss: No such file or directory
```

`npm install electron`
```
Error: GET https://github.com/electron/electron/releases/download/v4.0.5/electron-v4.0.5-linux-arm.zip returned 404
/home/pi/vscode/node_modules/electron/install.js:49
  throw err
  ^

Error: Failed to find Electron v4.0.5 for linux-arm at https://github.com/electron/electron/releases/download/v4.0.5/electron-v4.0.5-linux-arm.zip
```

The version is wrong; need `3.1.3` instead of `4.0.5`, and if we check 
https://github.com/electron/electron/releases/tag/v3.1.3

`npm install --arch=armv7l electron@v3.1.3`

```
> vscode-ripgrep@1.2.5 postinstall /home/pi/vscode/node_modules/vscode-ripgrep
> node ./lib/postinstall.js

Downloading ripgrep failed: Error: No asset named ripgrep-0.10.0-pcre-linux-armv7l.zip found
```
`npm install --arch=arm vscode-ripgrep@1.2.5`


`./scripts/npm.sh install --arch=armv7l`:
```
Error: Cannot find module '/usr/local/lib/nodejs/node-v10.15.1-linux-armv7l/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js'
```
`sudo npm install -g gyp`

`ELECTRON_MIRROR="https://npm.taobao.org/mirrors/electron/"`
https://github.com/Microsoft/vscode/wiki/How-to-Contribute#getting-the-sources

nodejs.org provides armv7 binaries and instructions:
https://nodejs.org/en/download/
https://github.com/nodejs/help/wiki/Installation

There's two other sources of pre-built binaries:
- https://node-arm.herokuapp.com/
- https://deb.nodesource.com


```
libGL error: No matching fbConfigs or visuals found
libGL error: failed to load driver: swrast
App threw an error during load
Error: Cannot find module 'minimist'
```

Ensure OpenGL driver is enabled:
1. `sudo raspi-config`
1. __Advanced > GL Driver > GL (full KMS)__

`npm install minimist --arch=armv7l`

```
{ errorCode: 'load',
  moduleId: 'iconv-lite',
  neededBy: [ 'vs/base/node/encoding' ],
  detail:
   { Error: Cannot find module 'iconv-lite'
   <SNIP CALLSTACK>
```

```
{ Error: Could not locate the bindings file. Tried:
 → /home/pi/projects/vscode/node_modules/spdlog/build/spdlog.node
 <SNIP>
 ```

```
(code-oss:12960): Gtk-WARNING **: cannot open display: localhost:10.0
```

Alternatively, there's pre-built binaries available from [http://code.headmelted.com/](http://code.headmelted.com/).