---
layout: post
title: Matrix Voice- A Microphone Array w/ Microcontroller
tags:
- iot
- embedded
- rust
series: Smart Home
---

It's also possible to do development without using PlatformIO.  See:

- [Github](https://github.com/matrix-io/matrixio_hal_esp32)
- [hackster post](https://www.hackster.io/matrix-labs/get-started-w-esp32-on-the-matrix-voice-d01e0d)


The [guide to install the stable release](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/index.html#installation-step-by-step) (there's also [latest/HEAD](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/index.html#installation-step-by-step)) is detailed and boils down to:

1. ESP32 toolchain/pre-requisites: [Mac](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/macos-setup.html), [Linux](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/linux-setup.html), [Windows](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/windows-setup.html)
1. Install ESP-IDF
    1. [Set `IDF_PATH`](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/add-idf_path-to-profile.html)
1. Install required python packages

https://github.com/platformio/platform-espressif32/releases/tag/v1.9.0 corresponds to v3.2.2.

On Mac/Linux:
```sh
# ESP32 toolchain/pre-requisites
sudo easy_install pip
mkdir -p ~/esp
cd ~/esp
curl -O https://dl.espressif.com/dl/xtensa-esp32-elf-osx-1.22.0-80-g6c4433a-5.2.0.tar.gz
tar -xzf xtensa-esp32-elf-osx-1.22.0-80-g6c4433a-5.2.0.tar.gz
# Add to PATH
export PATH=$HOME/esp/xtensa-esp32-elf/bin:$PATH

# Install ESP-IDF v3.2.2
git clone -b v3.2.2 --recursive https://github.com/espressif/esp-idf.git
# Set IDF_PATH
export IDF_PATH=$HOME/esp/esp-idf

# Install required python packages
python -m pip install --user -r $IDF_PATH/requirements.txt
```

```sh
git clone https://github.com/matrix-io/matrixio_hal_esp32.git
cd matrixio_hal_esp32/examples/everloop_demo

# Install bison
brew install bison
export PATH=/usr/local/opt/bison/bin:$PATH

make menuconfig
make -j4 # 4 jobs
export RPI_HOST=pi@raspberrypi.local
make deploy
```

https://www.hackster.io/matrix-labs/program-over-the-air-on-esp32-matrix-voice-w-arduino-ide-5e76bb

https://github.com/matrix-io/matrixio_hal_esp32/issues/9

https://docs.espressif.com/projects/esp-idf/en/latest/hw-reference/get-started-wrover-kit.html

## Rust

There's [an excellent write-up](https://dentrassi.de/2019/06/16/rust-on-the-esp-and-how-to-get-started/) of the work that has been done, but Rust support for ESP32 is hardly "mainstream": Xtensa/ESP32 support not yet in LLVM nor Rust mainlines.

```sh
git submodule update --init --recursive
docker run -it -v $(pwd):/home/matrix-io quay.io/ctron/rust-esp /bin/bash
# From https://github.com/ctron/rust-esp-container/blob/master/Dockerfile
#export LIBCLANG_PATH=/home/esp32-toolchain/llvm/llvm_install/
```

## SDK

https://rust-embedded.github.io/book/intro/install/macos.html
https://matrix-io.github.io/matrix-documentation/



`fatal error: 'sdkconfig.h' file not found`
```sh
# Generate sdkconfig
make menuconfig

# From: https://github.com/ctron/rust-esp-container/blob/master/Dockerfile
rustup toolchain link xtensa /home/esp32-toolchain/rustc/rust_build/
cargo install cargo-xbuild bindgen

./bindgen-project

# xbuild-project
cargo +xtensa xbuild --target "${XARGO_TARGET:-xtensa-esp32-none-elf}" --release

# image-project
"${IDF_PATH}/components/esptool_py/esptool/esptool.py" \
     --chip esp32 \
     elf2image \
     -o build/esp-app.bin \
     ../../target/xtensa-esp32-none-elf/release/everloop

# flash-project
"$IDF_PATH/components/esptool_py/esptool/esptool.py" \
     --chip esp32 \
     --port /dev/ttyS0 \
     --baud 115200 \
     --before default_reset \
     --after hard_reset \
     write_flash \
     -z \
     --flash_mode dio \
     --flash_freq 40m \
     --flash_size detect \
     0x1000 build/bootloader/bootloader.bin \
     0x10000 build/esp-app.bin \
     0x8000 build/partitions_singleapp.bin
```

`Error: CFI is not supported for this target` need `--release`

https://github.com/MabezDev/rust-xtensa/issues/5

https://github.com/espressif/llvm-project/issues/10

https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/log.html


```sh
# Install form, svd, and svd2rust
cargo install form
cargo install svd2rust
pip3 install --upgrade --user svdtools
```

If pip outputs the warning:
```
The script svd is installed in '/XXX/Python/3.7/bin' which is not on PATH.
```
Amend `PATH` environment variable:
```sh
export PATH=/XXX/Python/3.7/bin:$PATH
```
```
make
```