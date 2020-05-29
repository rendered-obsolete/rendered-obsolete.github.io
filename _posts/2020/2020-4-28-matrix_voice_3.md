---
layout: post
title: More Matrix Voice ESP32 in Rust
tags:
- iot
- embedded
- rust
series: Smart Home
---

Continuing with our [initial Rust work on the Matrix Voice]({% post_url /2020/2020-2-13-matrix_voice_rust %}), we've made significant progress.

## Toolchain

We [improved the Docker image](https://github.com/jeikabu/rust-esp-container/tree/master) containing LLVM/Rust binaries with Xtensa support:

- LLVM 9.0.1
- Rust 1.42
- [ESP-IDF v4.0](https://docs.espressif.com/projects/esp-idf/en/v4.0/get-started/index.html)

To build/run the Docker image:
```sh
git clone https://github.com/jeikabu/rust-esp-container
docker build -t rust-esp ./rust-esp-container
# Wait an hour or more
docker run --rm -it -v $(pwd):/home/matrix-io rust-esp /bin/bash
```

LLVM 9 brings support for debug information so it's no longer required to pass `--release` to cargo.

ESP-IDF v4.0 amongst other changes includes a new `idf.py` script and [CMake build system](https://docs.espressif.com/projects/esp-idf/en/v4.0/api-guides/build-system.html) in addition to the now "legacy" Make system:
```sh
# OLD
make -j4
# NEW
source $IDF_PATH/export.sh
idf.py build
```

While this may not seem like a ground breaking improvement, idf.py is a wrapper for much of the functionality that used to be different tools and scripts: make, esptool.py, etc.

To speed up build iterations compared to `build-project`, we introduced a `quick-build` script:
```sh
### OLD
# Build Rust binary: `target/xtensa-esp32-none-elf/`
# Using `[build].target` from `.cargo/config`:
cargo +xtensa xbuild
# OR, the more explicit (and longer):
cargo +xtensa xbuild --target "${XARGO_TARGET:-xtensa-esp32-none-elf}"
# Replace `build/esp-app.bin` with Rust binary
"${IDF_PATH}/components/esptool_py/esptool/esptool.py" \
    --chip esp32 \
    elf2image \
    -o build/esp-app.bin \
    target/xtensa-esp32-none-elf/CONFIG/PROJECT_NAME

### NEW
quick-build
```

The `create-project` script created some of the scaffolding required for a Rust project.  A native Rust feature has been [discussed for some time](https://internals.rust-lang.org/t/pre-rfc-cargo-templates/5056), but it seems [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) is the de-facto solution.  [esp_idf_template](https://github.com/jeikabu/esp_idf_template) takes care of the following:
- Default `sdkconfig` and makefiles to build native components with either CMake or "legacy" Make
- Cargo project with necessary `.cargo/config` and `build.rs` to build Rust binary that links with ESP-IDF
- Basic `Cargo.toml` and `main.rs` to get started

To use it:
```sh
cargo install cargo-generate
cargo generate -f --name PROJECT_NAME --git https://github.com/jeikabu/esp_idf_template.git --branch v4.0
```

[esp_idf_build](https://github.com/jeikabu/esp_idf_build) crate simplifies building Rust projects that use ESP-IDF:

- Previously, scripts for `esptool.py elf2image` explicitly specified the target ("xtensa-esp32-none-elf"), profile ("release"), and binary name ("esp32_everloop").  But `esp_idf_build::esptool_write_script()` generates `image.sh` which contains the correct values.

- `print_link_search()` sets the link search path for native libraries (via `println!("cargo:rustc-link-search=native={}")`) based on the `IDF_PATH` environment variable.  This makes the previous approach of a `esp-idf` symlink and lines like `"-C", "link-arg=-Lesp-idf/components/esp32/ld",` in `.cargo/config`  no longer necessary.

Care must be taken as a [bug in cargo makes mixing build scripts with cross-compiling occasionally unpredictable](https://github.com/rust-osdev/cargo-xbuild/issues/10).

Finally, we updated Rust [FFI bindings to ESP-IDF to v4.0](https://github.com/jeikabu/esp-idf-sys/tree/dev); adding the wifi, websocket, and mqtt APIs.

## Matrix Voice

The fine folks at Matrix-io started a re-write of their C++ HAL in Rust.  We contributed [an implementation for the Voice w/ ESP32](https://github.com/jeikabu/matrix-rhal/tree/dev) via ESP-IDF.  This includes an [implementation of the "everloop" demo in Rust](https://github.com/jeikabu/matrix-rhal/tree/no_std/examples/esp32_everloop):

```rust
#![no_std]
#![no_main]

extern crate matrix_rhal;

use matrix_rhal::bus::MatrixBus;

#[no_mangle]
pub fn app_main() {
    let bus = matrix_rhal::bus::init();
    let everloop = matrix_rhal::Everloop::new(&bus);
    let mut counter = 0;
    loop {
        const NUMBER_LEDS: usize =
            matrix_rhal::bus::memory_map::device_info::MATRIX_VOICE_LEDS as usize;
        let mut image1d = [matrix_rhal::Rgbw::black(); NUMBER_LEDS];
        image1d[(counter / 2) % NUMBER_LEDS].r = 20;
        image1d[(counter / 7) % NUMBER_LEDS].g = 30;
        image1d[(counter / 11) % NUMBER_LEDS].b = 30;
        image1d[NUMBER_LEDS - 1 - (counter % NUMBER_LEDS)].w = 10;
        everloop.set(&image1d);

        counter += 1;
        unsafe {
            // Set RTC timer to trigger wakeup and then enter light sleep
            esp_idf_sys::esp_sleep_enable_timer_wakeup(25000);
            esp_idf_sys::esp_light_sleep_start();
        }
    }
}

extern "C" {
    fn abort() -> !;
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    unsafe {
        abort();
    }
}
```

The last part of `app_main()` is interesting as it makes use of ESP32 [sleep modes](https://docs.espressif.com/projects/esp-idf/en/v4.0/api-reference/system/sleep_modes.html).  Rather than a simple delay or "thread sleep" there's "light" and "deep" sleep power-saving modes:

| Mode | Powered Off | Powered On |
|-|-|-
| Light | Wifi/BT | CPUs/RAM/peripherals clock-gated, supply voltage reduced
| Deep | CPUs/RAM/peripherals/Wifi/BT | RTC controller/peripherals/memories

The API provides a number of options such as what triggers wake-up; here we're using RTC, but you could instead use GPIO, etc.  It's also possible to [put the wifi modem to sleep](https://github.com/espressif/esp-idf/tree/release/v4.0/examples/wifi/power_save) so an AP connection is maintained.

## Niceties

Using the native `ets_printf` for logging was pretty painful:
```rust
unsafe {
    esp_idf_sys::ets_printf(b"value=%d\n\0".as_ptr() as *const _, 1);
}
```

We added [esp_idf_logger, an initial ESP-IDF implementation](https://github.com/jeikabu/esp_idf_logger) of the [log facade](https://crates.io/crates/log).

The implementation for `fn log(&self, record: &log::Record)` had us stumped for a while.  Particularly because the return value of [`log::Record::args() -> core::fmt::Arguments`](https://docs.rs/log/0.4.10/log/struct.Record.html#method.args) as you get from [`format_args!`](https://doc.rust-lang.org/nightly/std/macro.format_args.html), is normally passed to `format!`.  However, `format!` is in libstd and not available in no_std.

[This SO](https://stackoverflow.com/questions/39488327/how-to-format-output-to-a-byte-array-with-no-std-and-no-allocator) provides one solution.  Basically, you `impl core::fmt::Write` for a type that wraps the buffer, then call `write!(wrapper, "{}", record.args())`:

```rust
struct Wrapper<'a> {
    buf: &'a mut [u8],
    offset: usize,
}

impl<'a> fmt::Write for Wrapper<'a> {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        // Append `s` to `self.buf[offset]`
    }
}

/// Elsewhere

// C API expects log output ends with newline and nul-terminating 0
let buffer_suffix: &[u8; SUFFIX_SIZE] = b"\n\0";
let mut string_buffer = [0u8; BUFFER_SIZE];
// Always leave enough space for `\n\0`
let mut writer = Wrapper::new(&mut string_buffer[..BUFFER_SIZE - SUFFIX_SIZE]);
let res = write!(writer, "{}", record.args());
let mut offset = writer.offset();

// Error handling

// Append `\n\0`
string_buffer[offset..offset + SUFFIX_SIZE].copy_from_slice(buffer_suffix);
unsafe {
    esp_idf_sys::ets_printf(string_buffer.as_ptr() as *const _);
}
```

Now you can write the very svelte:
```rust
esp_idf_logger::init().unwrap();
log::info!("value={}", 1);
```

Finally, [esp_idf](https://github.com/jeikabu/esp_idf) is a (_very incomplete_) high-level wrapper for esp-idf-sys.  It hides some of the `unsafe`, pointer, naming, and usage nastiness of the bindgen generated FFI bindings and replaces functions that return a C int with a more natural `Result`.  This is shown in more detail below.

## Wifi

One of our highest priority goals is to get the Voice/ESP32 on wifi.  Starting with a direct port of the ESP-IDF [wifi example](https://github.com/espressif/esp-idf/tree/v4.0/examples/wifi):

```rust
include!("config.rs");

#[no_mangle]
pub fn app_main() {
    esp_idf_logger::init().unwrap();

    wifi().unwrap();

    log::info!("OK");
}

fn wifi() -> Result<(), esp_idf::error::Error> {
    use esp_idf::*;

    nvs::init()?;
    tcpip::init();
    event::loop_create_default()?;
    let cfg = wifi::InitConfig::default();
    wifi::init(cfg)?;
    event::handler_register(event::events::ip::StaGotIp, event_handler)?;
    event::handler_register(event::events::wifi::Any, event_handler)?;

    let config = wifi::StaConfig::from(&CONFIG).unwrap();
    wifi::set_mode(wifi::WifiMode::STA)?;
    wifi::set_sta_config(config)?;
    wifi::start()
}
```

In particular, eschewing C API naming convention for modules and replacing `int` return values with `Result`:
```rust
// In esp-idf-sys/bindings.rs
extern "C" {
    pub fn esp_event_loop_create_default() -> esp_err_t;
}
// In esp_idf/
mod event {
    pub fn loop_create_default() -> Result<(), error::Error> {
        unsafe {
            let retval = esp_event_loop_create_default();
            esp_int_into_result(retval)
        }
    }
}
```

Another goal was to discourage committing AP passwords into SCC.  Here using `include!` to bring `.gitignore`'d file `config.rs` in at compile time:
In `config.rs`:
```rust
const CONFIG: Config = Config {
    ssid: "YYYYYYYYYY",
    password: "XXXXXXXXXXXX",
};
```

The next step will be to have `build.rs` extract the password from an environment variable.

ESP-IDF events are handled by:
```rust
unsafe extern "C" fn event_handler(
    _event_handler_arg: *mut core::ffi::c_void,
    event_base: esp_event_base_t,
    event_id: i32,
    _event_data: *mut core::ffi::c_void,
) {
    use esp_idf::event::*;
    match Event::try_from((event_base, event_id)) {
        Ok(Event::Wifi(WifiEvent::StaStart)) | Ok(Event::Wifi(WifiEvent::StaDisconnected)) => {
            log::info!("esp_wifi_connect trying...");
            esp_idf::wifi::connect().unwrap();
            log::info!("esp_wifi_connect ok.");
        },
        Ok(Event::Ip(IpEvent::StaGotIp)) => log::info!("GOT_IP!"),
        Err(_) => log::warn!("Unhandled event: {:?} {}", EventBase::try_from(event_base), event_id),
        _ => {},
    }
}
```

There are improvements to make this more ergonomic (including [support for Rust closures](https://blog.seantheprogrammer.com/neat-rust-tricks-passing-rust-closures-to-c)).  As well as upcoming [pattern matching improvements](https://blog.rust-lang.org/inside-rust/2020/03/04/recent-future-pattern-matching-improvements.html):

```rust
// OLD
match Event::try_from((event_base, event_id)) {
        Ok(Event::Wifi(WifiEvent::StaStart)) | Ok(Event::Wifi(WifiEvent::StaDisconnected)) => { },
}
// NEW
match Event::try_from((event_base, event_id)) {
        Ok(Event::Wifi(WifiEvent::StaStart | WifiEvent::StaDisconnected)) => { },
}
```

## Binary Size

At one point during development we hit the limits of the flash-able binary size.  This drove an investigation into Rust binary sizes.  There's several good resources:

- https://github.com/johnthagen/min-sized-rust
- https://rust-embedded.github.io/book/unsorted/speed-vs-size.html
- https://jamesmunns.com/blog/tinyrocket/

Make sure to look at the binary produced by `esptool.py elf2image` because this will strip debug symbols from the ELF binary and is a more accurate representation of what you'll be flashing.

### `lto`

In `Cargo.toml` you can specify a number of [per-profile options](https://doc.rust-lang.org/cargo/reference/profiles.html) including [link-time optimization (LTO)](https://en.wikipedia.org/wiki/Interprocedural_optimization):
```toml
[profile.dev]
lto = "fat"
incremental = false # if enabled, `codegen-units` is ignored
codegen-units = 1

[profile.release]
lto = "fat"
codegen-units = 1
```

| | Dev | Release | Notes
|-|-|-|-
| Initial | 629472 | 618896 | 
| lto = "thin" | 633056 | 618784 | The docs say "thin" should be almost as good and faster to build
| lto = "fat" | 635168 | 610224 | Same as `lto = true`
| fat + codegen-units = 1 | 633088 | 609632

## `panic="abort"`

| | Dev | Release | Notes
|-|-|-|-
| Initial | 629472 | 618896 | 
| panic="abort"| 629488 | 618928

Several online resources mention this, but it either no longer works or is ineffective in no_std environments.

## `opt-level="z"`

The compiler can be instructed to optimize for size (vs speed).

There's two different ways this can be done:
- For dependencies only via [overrides](https://doc.rust-lang.org/cargo/reference/profiles.html#overrides)
- For everything

```toml
# Dependencies
[profile.dev.package."*"]
opt-level = "z"

[profile.release.package."*"]
opt-level = "z"

# Everything
[profile.dev]
opt-level = "z"

[profile.release]
opt-level = "z"
```

This can be used to, for example, keep the top-level crate debug-able while reducing the size of dependencies.

| | Dev | Release | Notes
|-|-|-|-
| Initial | 629472 | 618896 | 
| dependencies | 620128 | 618256
| opt-level="z" | 617632 | 617616

Not terribly surprising since this project has few dependencies.

## `menuconfig`

Running either `make menuconfig` or `idf.py menuconfig` you can tweak numerous values set in `sdkconfig`.  Scouring ESP-IDF forums reveals numerous suggestions:
- In __Compiler Options__, set __Optimization Level>Release__ and __Assertion level>Silent__
- In __Component config>Newlib>Enable 'nano' formatting options__

| | Dev | Release | Notes
|-|-|-|-
| Initial | 629472 | 618896 | 
| menuconfig | 536928 | 526336 | 

This yields the biggest wins.  Not entirely surprising given that ESP-IDF represents the bulk of the code base.  Disabling unneeded components and features like older versions of TLS can yield further gains depending on your requirements.

Note that "nano formatting options" requires changes in `.cargo/config`:
```
"-C", "link-arg=-Tesp32.rom.newlib-nano.ld",
# Replace
#"-C", "link-arg=-lc",
# With
"-C", "link-arg=-lc_nano",
```

## Summary

Forming up like Voltron, combined they yield:

| | Dev | Release | Notes
|-|-|-|-
| Initial | 629472 | 618896 | 
| All the above | 525376 | 516896

It's clear menuconfig has the potential to provide the biggest wins.  We can investigate that further.

In `.cargo/config` add:
`"-C", "link-arg=-Wl,-Map=build/esp-app.map",`

`idf.py size` displays a short summary:
```
Total sizes:
 DRAM .data size:   12252 bytes
 DRAM .bss  size:   24336 bytes
Used static DRAM:   36588 bytes ( 144148 available, 20.2% used)
Used static IRAM:   79880 bytes (  51192 available, 60.9% used)
      Flash code:  353517 bytes
    Flash rodata:   71132 bytes
Total image size:~ 516781 bytes (.bin may be padded larger)
```

`idf.py size-components` adds per-library details (only showing top ten):
```
Per-archive contributions to ELF file:
            Archive File DRAM .data & .bss   IRAM Flash code & rodata   Total
           libnet80211.a        925   8884  11343     109814    21466  152432
               liblwip.a         21   3990      0      68557    17134   89702
                 libpp.a       1282   5314  23480      37600     5048   72724
                libphy.a       1604    929   6491      30262        0   39286
     libwpa_supplicant.a          0    686      0      30000     4377   35063
           libfreertos.a       4140    776  12869          0     1289   19074
             libc_nano.a        364      0      0      16131      512   17007
              libesp32.a       2314    133   6396       5311     2182   16336
          libnvs_flash.a          0     32      0       9939      266   10237
                libsoc.a        132      4   5854        676     3479   10145
```

`idf.py size-files` adds info from individual object files (again, top offenders):
```
Per-file contributions to ELF file:
             Object File DRAM .data & .bss   IRAM Flash code & rodata   Total
       ieee80211_ioctl.o        620    403      0      18159     3509   22691
                wl_cnx.o          2     51    262      17902     4384   22601
      ieee80211_output.o          3   2600   3450      11744     1166   18963
                    pp.o        180    917  10151       5570      716   17534
           phy_chip_v7.o        921    827   1029      14219        0   16996
         ieee80211_sta.o          0     34   6174       6286     2338   14832
                  wdev.o         51     54   6146       7097      676   14024
        ieee80211_scan.o         14    284      0      10723     2656   13677
                   trc.o        636   1904   3493       5487     1271   12791
       phy_chip_v7_cal.o        477     53   3420       8711        0   12661
                  lmac.o          3    234   1948       9320      715   12220
          ieee80211_ht.o          4      4   1383       8938     1114   11443
      ieee80211_hostap.o          1     41      0       9459      735   10236
```

Looks like we could shed another 200KB or so by dropping networking support.  But then it wouldn't make for a very good wifi demo.

Also wanted to give a Rust-specific tool [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) a try.  It didn't work, but can be installed with:
```sh
cargo install cargo-bloat --features regex-filter
```

