---
layout: post
title: Matrix Voice ESP32 in Rust
tags:
- iot
- embedded
- rust
- bindgen
series: Smart Home
---

My ultimate goal for [Matrix Voice w/ ESP32]({% post_url /2020/2020-1-6-matrix_voice %}) is to be able to develop in Rust.

## Basic Toolchain

Previously we used their recommend dev environment of PlatformIO.  It's also possible to do development without using PlatformIO.  Matrix-io provides [matrixio_hal_esp32] C/C++ repository for programming of Matrix Voice with ESP32.  The [Github repo][matrixio_hal_esp32] and this [hackster post](https://www.hackster.io/matrix-labs/get-started-w-esp32-on-the-matrix-voice-d01e0d) contain details.

[From the initial tests]({% post_url /2020/2020-1-6-matrix_voice %}#serial-output) we know Matrix Voice depends on release `v1.9.0` of platformio's espressif toolchain.  Looking at [platformio releases](https://github.com/platformio/platform-espressif32/releases/tag/v1.9.0) this corresponds to ESP-IDF v3.2.2.

The [guide to install v3.2.2](https://docs.espressif.com/projects/esp-idf/en/v3.2.2/get-started/index.html) (there's also [stable](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/index.html#installation-step-by-step) as well as [latest/HEAD](https://docs.espressif.com/projects/esp-idf/en/latest/get-started/index.html#installation-step-by-step)) is detailed and boils down to:

1. ESP32 toolchain/pre-requisites (for [Mac](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/macos-setup.html), [Linux](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/linux-setup.html), [Windows](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/windows-setup.html))
1. Install ESP-IDF and [set `IDF_PATH`](https://docs.espressif.com/projects/esp-idf/en/stable/get-started/add-idf_path-to-profile.html)
1. Install required python packages


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

Build and run [matrixio_hal_esp32](https://github.com/matrix-io/matrixio_hal_esp32) example:
```sh
git clone https://github.com/matrix-io/matrixio_hal_esp32.git
cd matrixio_hal_esp32/examples/everloop_demo

# Mac Only: Install bison
brew install bison
export PATH=/usr/local/opt/bison/bin:$PATH

make menuconfig
make -j4
export RPI_HOST=pi@raspberrypi.local
make deploy
```

The deploy script currently requires a small hack to work correctly (see [this issue](https://github.com/matrix-io/matrixio_hal_esp32/issues/9)).


## Rust-ified

There's [an excellent write-up](https://dentrassi.de/2019/06/16/rust-on-the-esp-and-how-to-get-started/) of the current state of ESP32 development in Rust:
- [LLVM support for Xtensa/ESP32](https://github.com/espressif/llvm-project) (not yet) in mainline
- [Rust support](https://github.com/MabezDev/rust-xtensa) also not in mainline (yet)
- Need [`no_std`-i.e. excluding the standard library](https://doc.rust-lang.org/1.7.0/book/no-stdlib.html)

Building and installing custom versions of LLVM/Rust along with configuring cross-compiling is a bit onerous, but the author [put together a docker image](https://github.com/ctron/rust-esp-container):
```sh
git clone https://github.com/ctron/rust-esp-container.git
cd rust-esp-container
git submodule update --init --recursive
cd ../matrix-io
docker run -it -v $(pwd):/home/matrix-io quay.io/ctron/rust-esp /bin/bash
```

## matrix_hal_esp32_sys

The [`components/hal/` directory of matrix_hal_esp32](https://github.com/matrix-io/matrixio_hal_esp32/tree/master/components/hal) contains C++ to access Matrix Voice-specific functionality.  Let's pass it through [bindgen](https://github.com/rust-lang/rust-bindgen) to generate a Rust FFI wrapper `matrix_hal_esp32_sys`.

With a [basic `Makefile`](https://github.com/jeikabu/matrix-io/blob/master/matrix_hal_esp32_sys/Makefile):
```makefile
PROJECT_NAME := esp-app

EXTRA_COMPONENT_DIRS += $(PROJECT_PATH)/matrixio_hal_esp32/components

include $(IDF_PATH)/make/project.mk
include $(PROJECT_PATH)/matrixio_hal_esp32/make/deploy.mk
```

[This `bindgen-project` script](https://github.com/jeikabu/matrix-io/blob/master/matrix_hal_esp32_sys/bindgen-project) based off [the original](https://github.com/ctron/rust-esp-container/blob/master/bindgen-project).

[In `build.rs`](https://github.com/jeikabu/matrix-io/blob/master/matrix_hal_esp32_sys/build.rs):
```rs
use std::{env, path::PathBuf};

fn main() {
    link_nng();
    bindgen_generate();
}

fn link_nng() {
    let root = PathBuf::from(env::var("CARGO_MANIFEST_DIR").unwrap());
    println!(
        "cargo:rustc-link-search=native={}", root.join("build/hal/").display()
    );
    // Link to matrixio_hal_esp32 generated `hal` library
    println!("cargo:rustc-link-lib=static=hal");
}
```

The main thing being linking to `build/hal/libhal.a` generated by `make`:

```sh
# Generate sdkconfig
make menuconfig
# Populate `build/`
make -j4

# From: https://github.com/ctron/rust-esp-container/blob/master/Dockerfile
export LIBCLANG_PATH=/home/esp32-toolchain/llvm/llvm_install/
rustup toolchain link xtensa /home/esp32-toolchain/rustc/rust_build/
cargo install cargo-xbuild bindgen

./bindgen-project

# From `xbuild-project` script
cargo +xtensa xbuild --target "${XARGO_TARGET:-xtensa-esp32-none-elf}" --release
```

Make sure to build `--release` of you'll get the mysterious error: `Error: CFI is not supported for this target`.  Debug configuration isn't yet supported, but should be shortly (see issues [#1](https://github.com/MabezDev/rust-xtensa/issues/5) and [#2](https://github.com/espressif/llvm-project/issues/10)).


## "Hello World"

esp-idf contains [several `esp_log_*` funtions](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/log.html), but they don't work here.  There's also [several `ESP_EARLY_LOGx` macros](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/log.html#c.ESP_EARLY_LOGE) which [ultimately call](https://github.com/espressif/esp-idf/blob/master/components/log/include/esp_log.h#L268) `ets_printf()` enabling you to write "hello world":
```rust
#![no_std]
#![no_main]

#[no_mangle]
pub fn app_main() {
    unsafe {
        use matrix_hal_esp32_sys::*;
        // `b`-prefix creates byte string, `\0` null-terminates it
        let text = b"Hello World\n\0";
        // `ets_printf()` takes a null-terminated `* const u8`
        ets_printf(text.as_ptr() as *const _);
    }
}

use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

- `#![no_std]` precludes using the standard library (see [embedded book](https://rust-embedded.github.io/book/intro/no-std.html))
- `#![no_main]` lets us build a binary without `main()`- we instead have `app_main()`
- `#[no_mangle]` ensures `app_main()` symbol isn't mangled in way unreconizable to native linkers (see [nomicon](https://doc.rust-lang.org/nomicon/ffi.html) and [embedded](https://rust-embedded.github.io/book/interoperability/rust-with-c.html))
- `#[panic_handler]` handles `panic!()` in `no_std` applications (see [nomicon](https://doc.rust-lang.org/nomicon/panic-handler.html) and [reference](https://doc.rust-lang.org/reference/runtime.html))


```sh
# From `image-project` script
"${IDF_PATH}/components/esptool_py/esptool/esptool.py" \
     --chip esp32 \
     elf2image \
     -o build/esp-app.bin \
     ../../target/xtensa-esp32-none-elf/release/everloop
```

[`install.sh`](https://github.com/jeikabu/matrix-io/blob/master/examples/everloop/install.sh) comes from [esp32-platformio](https://github.com/matrix-io/esp32-platformio/blob/master/ota/install.sh), and is basically identical to [`flash-project` in docker](https://github.com/ctron/rust-esp-container/blob/master/flash-project) using `esp_tool` to push the build to the device.

Previously we used [screen to monitor console output]({% post_url /2020/2020-1-6-matrix_voice %}#serial-output) from the ESP32, but `tail` is better since we can still write images to the serial port white capturing output:
```sh
# On Raspberry Pi:
tail -f /dev/ttyS0
# OR
screen /dev/ttyS0 115200
```

## Everloop

Matrix-io provides an [example `everloop_demo`](https://github.com/matrix-io/matrixio_hal_esp32/tree/master/examples/everloop_demo).

The C++:
```cpp
#include <stdio.h>
#include <cmath>

#include "esp_system.h"

#include "everloop.h"
#include "everloop_image.h"
#include "voice_memory_map.h"
#include "wishbone_bus.h"

namespace hal = matrix_hal;

int cpp_loop() {
  hal::WishboneBus wb;

  wb.Init();

  hal::Everloop everloop;
  hal::EverloopImage image1d;

  everloop.Setup(&wb);

  unsigned counter = 0;
  int blue = 0;

  while (1) {
    // Pulsing blue
    blue = static_cast<int>(std::sin(counter / 64.0) * 10.0) + 10;
    for (hal::LedValue& led : image1d.leds) {
      led.red = 0;
      led.green = 0;
      led.blue = blue;
      led.white = 0;
    }
    // Set the LEDs
    everloop.Write(&image1d);
    ++counter;
  }

  return 0;
}

extern "C" {
  // Entry point
  void app_main(void) { cpp_loop(); }
}
```

If you've seen [any of the]({% post_url /2019/2019-09-30-lmbr_rust %}) [Rust/Lumberyard]({% post_url /2019/2019-10-05-lmbr_editor_rust %}) [stuff]({% post_url /2019/2019-12-06-lmbr_ebus %}) I've been experimenting with, or you've tried on your own codebases, you're familiar with bindgen's limitations regarding C++ and especially [STL](http://www.cplusplus.com/reference/stl/).

```cpp
//----------------------------------------------------
// everloop_image.h

const int kMatrixCreatorNLeds = 18;

// An array of 18 LED values
class EverloopImage {
 public:
  EverloopImage(int nleds = kMatrixCreatorNLeds) { leds.resize(nleds); }
  std::vector<LedValue> leds;
};

//----------------------------------------------------
// everloop.cpp

bool Everloop::Write(const EverloopImage* led_image) {
  if (!wishbone_) return false;

  // Create array of 18*4 bytes
  std::valarray<unsigned char> write_data(led_image->leds.size() * 4);

  // Fill RGB-White values
  uint32_t led_offset = 0;
  for (const LedValue& led : led_image->leds) {
    write_data[led_offset + 0] = led.red;
    write_data[led_offset + 1] = led.green;
    write_data[led_offset + 2] = led.blue;
    write_data[led_offset + 3] = led.white;
    led_offset += 4;
  }

  // Write array of values to wishbone bus
  wishbone_->SpiWrite(kEverloopBaseAddress, &write_data[0], write_data.size());
  return true;
}
```

We can write a pure Rust version of most of this and skip wrangling with what bindgen generates from C++.

[The `Cargo.toml`](https://github.com/jeikabu/matrix-io/blob/master/examples/everloop/Cargo.toml):
```toml
[package]
name = "everloop"
version = "0.1.0"
edition = "2018"

[dependencies]
matrix_hal_esp32_sys = {path = "../../matrix_hal_esp32_sys"}
# `no_std` access to math functions like `sin()`
libm = "0.2"
# C types for FFI
cty = {version = "0.2"}
```

Owing to the `no_std` requirement: [libm](https://crates.io/crates/libm) gives us access to [math routines like `sin()`](https://doc.rust-lang.org/std/primitive.f32.html#method.sin), and [cty](https://crates.io/crates/cty) provides [`std::os::raw` types](https://doc.rust-lang.org/std/os/raw/index.html).

In Rust:
```rust
#![no_std]
#![no_main]

// Entry point
#[no_mangle]
pub fn app_main() {
    unsafe {
        everloop();
    }
}

unsafe fn everloop() {
    use matrix_hal_esp32_sys::*;
    let mut wb = matrix_hal::WishboneBus::default();
    wb.Init();

    // Don't bother with Everloop helper class, it just makes a byte array
    // let mut everloop = matrix_hal::Everloop::new();
    // everloop._base.Setup(&mut wb);

    let mut counter = 0;
    loop {
        const NUMBER_LEDS: usize = matrix_hal::kMatrixCreatorNLeds as usize;
        let mut image1d = [0u8; NUMBER_LEDS * 4];
        let blue = (libm::sinf(counter as f32 / 64.0) * 10.0 + 10.0) as u8;
        ets_printf(b"counter=%d blue=%d\n\0".as_ptr() as *const _, counter, blue as cty::c_uint);
        for i in 0..NUMBER_LEDS {
            image1d[i * 4 + 2] = blue;
        }
        wb.SpiWrite(matrix_hal::kEverloopBaseAddress as u16, image1d.as_ptr(), image1d.len() as i32);
        counter += 1;
    }
}

// Same panic handler as above
#[panic_handler]
//...
```

## Tips

[Remote Development](https://code.visualstudio.com/docs/remote/remote-overview) has become one of my favorite plugins for VS Code.  It lets you run a VS Code session locally that interacts with a remote/virtual environment.  [Remote- SSH](https://code.visualstudio.com/docs/remote/ssh) makes it easier to work with another device like a Raspberry Pi- and is decidedly better than X11 forwarding.  [Remote- Containers](https://code.visualstudio.com/docs/remote/containers) lets you do the same with running containers and is especially handy when you want to mess with files not in a volume mounted from the host.

## Next

- Potentially replace matrix_hal_esp32_sys with pure Rust implementation of matrix_hal_esp32 via [esp-idf-sys](https://github.com/sapir/esp-idf-sys) or [esp-sys](https://github.com/esp-rs/esp32)
- Replace direct use of `ets_printf` with [logger implementation](https://docs.rs/log/0.4.10/log/)
- Better understand what's going on with `.cargo/config` and `cargo-xbuild` and move as much as possible to [Rust `build.rs` build script](https://doc.rust-lang.org/cargo/reference/build-scripts.html)


[matrixio_hal_esp32]: https://github.com/matrix-io/matrixio_hal_esp32
