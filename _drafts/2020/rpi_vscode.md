## Visual Studio Code

[This post](https://www.hanselman.com/blog/BuildingVisualStudioCodeOnARaspberryPi3.aspx) seems to be the standard source for building VS Code.  In general you need to:

1. Download [latest release](https://github.com/Microsoft/vscode/releases)
1. Follow [instructions to build and run](https://github.com/Microsoft/vscode/wiki/How-to-Contribute#build-and-run)

You'll run into problems with `yarn` if you blindly attempt `apt-get install yarn` you end up with `cmdtest`.  Following the [instructions from yarnpkg.com](https://yarnpkg.com/en/docs/install#debian-stable):
```bash
# Remove it "just in case"
sudo apt-get remove cmdtest
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
```

Install yarn and other VS Code pre-requisites:
```bash
sudo apt-get update
sudo apt-get install -y yarn libx11-dev libsecret-1-dev libxkbfile-dev
```

If you `apt-get install` nodejs the version you get is too old:
1. Check version `node --version`
1. Remove it `sudo apt-get remove nodejs`
1. Find the version from [nodejs downloads](https://nodejs.org/en/download/releases/)
1. Download the `linux-armv7l` tarball
1. [Install nodejs binary](https://github.com/nodejs/help/wiki/Installation)
    1. Uncompress:
        ```bash
        sudo mkdir -p /usr/local/lib/nodejs
        # xJf if you downloaded the .tar.xz
        sudo tar xzf node-v8.15.1-linux-armv7l.tar.gz -C /usr/local/lib/nodejs
        ```
    1. Create links:
        ```bash
        cd /usr/bin && sudo ln -s /usr/local/lib/nodejs/node-v8.15.1-linux-armv7l/bin/node node \
        && sudo ln -s /usr/local/lib/nodejs/node-v8.15.1-linux-armv7l/bin/npm npm \
        && sudo ln -s /usr/local/lib/nodejs/node-v8.15.1-linux-armv7l/bin/npx npx
        ```

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

`export ELECTRON_MIRROR="https://npm.taobao.org/mirrors/electron/"`
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

Launch VS Code:
```

```


# Second

In theory you need to:
```bash
# Install dependencies
yarn
# Build
yarn watch
```

You'll see:  
![](/assets/vscode_finished_compilation.png)

To run VS Code:
```bash
# 
./scripts/code.sh
```

I repeatedly had issues with ripgrep:
```
error /home/pi/Downloads/vscode-1.31.1/node_modules/vscode-ripgrep: Command failed.
Exit code: 1
Command: node ./lib/postinstall.js
Arguments: 
Directory: /home/pi/Downloads/vscode-1.31.1/node_modules/vscode-ripgrep
Output:
Downloading to /tmp/vscode-ripgrep-cache-1.2.5/ripgrep-0.10.0-pcre-linux-arm.zip
Downloading ripgrep-0.10.0-pcre-linux-arm.zip...
events.js:183
      throw er; // Unhandled 'error' event
      ^
```
https://github.com/Microsoft/vscode/issues/61164

Also during `yarn` downloading electron would timeout:
```
error /home/pi/Downloads/vscode-1.31.1/test/smoke/node_modules/electron: Command failed.
Exit code: 1
Command: node install.js
Arguments: 
Directory: /home/pi/Downloads/vscode-1.31.1/test/smoke/node_modules/electron
Output:
Downloading tmp-32148-0-electron-v3.1.2-linux-armv7l.zip
Error: read ECONNRESET
/home/pi/Downloads/vscode-1.31.1/test/smoke/node_modules/electron/install.js:49
  throw err
  ^
```

Just keep running `yarn` until it completes.  Probably need to find a find a different npm source.

I said "in theory" because I never actually saw `yarn watch` complete...

`gulp` memory usage will slowly climb, sometime after `Starting 'watch-client'...` it will hit around 90% and `kswapd` will become one of the top users of CPU and memory and the system will become mostly unresponsive.

While you let that wring your Pi into Raspberry compote (or chutney, if that's your fancy), let's look at cross-compiling:
https://github.com/Microsoft/vscode/wiki/Cross-Compiling-for-Debian-Based-Linux

```bash
sudo apt-get install -y fakeroot rpm
```

Replace `bionic` with the version of Ubuntu you're running:
```
sudo qemu-debootstrap --arch=armhf --variant=minbase bionic rootfs
sudo chroot rootfs apt-get install -y libx11-dev libsecret-1-dev libxkbfile-dev
```

```
make: Entering directory '/home/jake/Downloads/vscode-1.31.1/node_modules/native-watchdog/build'
  CXX(target) Release/obj.target/watchdog/src/watchdog.o
  SOLINK_MODULE(target) Release/obj.target/watchdog.node
/usr/lib/gcc-cross/arm-linux-gnueabihf/7/../../../../arm-linux-gnueabihf/bin/ld: cannot find /lib/arm-linux-gnueabihf/libpthread.so.0
/usr/lib/gcc-cross/arm-linux-gnueabihf/7/../../../../arm-linux-gnueabihf/bin/ld: cannot find /usr/lib/arm-linux-gnueabihf/libpthread_nonshared.a
collect2: error: ld returned 1 exit status
watchdog.target.mk:123: recipe for target 'Release/obj.target/watchdog.node' failed
make: *** [Release/obj.target/watchdog.node] Error 1
```


No yarn watch:
```
./scripts/code.sh: line 50: /home/pi/Downloads/vscode-1.31.1/.build/electron/code-oss: No such file or directory
```

```
sudo apt-get update && \
sudo apt-get install -y docker.io && \
wget https://github.com/Microsoft/vscode/archive/1.32.1.tar.gz && \
tar xzf 1.32.1.tar.gz && \
sudo docker run --rm --privileged multiarch/qemu-user-static:register --reset

sudo docker run -it -v $(pwd)/vscode-1.32.1:/usr/src multiarch/debian-debootstrap:armhf-stretch

export NODE_VER=v8.15.1 && \
export PYTHON=python2.7 && \
apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    vim && \
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" > /etc/apt/sources.list.d/yarn.list && \
apt-get update && apt-get install -y \
    build-essential \
    fakeroot \
    libx11-dev libsecret-1-dev libxkbfile-dev \
    python2.7 \
    rpm \
    yarn && \
ln -s `which python2.7` /usr/bin/python && \
curl -sS https://nodejs.org/download/release/$NODE_VER/node-$NODE_VER-linux-armv7l.tar.xz > nodejs.tar.xz \
    && mkdir -p /usr/local/lib/nodejs \
    && tar xJf nodejs.tar.xz -C /usr/local/lib/nodejs \
    && ln -s /usr/local/lib/nodejs/node-$NODE_VER-linux-armv7l/bin/node /usr/bin/node \
    && ln -s /usr/local/lib/nodejs/node-$NODE_VER-linux-armv7l/bin/npm /usr/bin/npm \
    && ln -s /usr/local/lib/nodejs/node-$NODE_VER-linux-armv7l/bin/npx /usr/bin/npx