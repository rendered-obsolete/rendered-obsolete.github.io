---
layout: post
title: Early Raspberry Pi Pico and Rust
tags:
- iot
- embedded
- raspberrypi
- rust
- pico
---

The [Raspberry Pi Pico](https://www.raspberrypi.org/products/raspberry-pi-pico/) announced earlier this year is a $4 [microcontroller](https://en.wikipedia.org/wiki/Microcontroller) board from the makers of the ever-popular Raspberry Pi.  As a Rust-enthusiast one of first questions I tend to investigate is:

[_Can it run Rust?_](https://www.google.com/search?q=can+it+run+crysis)

__TL;DR__ Yes, but it's not quite ready for prime-time.  _Yet_.

## Hardware

The Pico utilizes the (also new) RP2040 microcontroller.  Compared to the already minimalist Pi Zero, it has even more modest [specs](https://www.raspberrypi.org/documentation/rp2040/getting-started/#board-specifications):

- Dual-core Arm Cortex M0+ processor at up to 133 MHz
- 264KB of SRAM, and 2MB of on-board Flash memory
- USB 1.1 with device and host support
- 26 × multi-function GPIO pins
- 2 × SPI, 2 × I2C, 2 × UART, 3 × 12-bit ADC, 16 × controllable PWM channels
- Temperature sensor
- Accelerated floating-point libraries on-chip
- 8 × Programmable I/O (PIO) state machines for custom peripheral support

## Rust Demo

First, download and install [ARM toolchain for the host system](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads).  If necessary, add it to `PATH` environment variable.  For example, if you install the `.pkg` on a Mac you'll need to add `/Applications/ARM/bin/`.

Next, setup rest of build environment and build a simple demo:

```sh
# Add support for ARM Cortex M0+
rustup target add thumbv6m-none-eabi

# Build pico-sdk for elf2uf2 binary
git clone https://github.com/raspberrypi/pico-sdk --depth 1 --recursive --single-branch
pushd pico-sdk && mkdir build && cd build
cmake .. && make
# Output: build/elf2uf2/elf2uf2 (executable)
popd

# Build sample program and convert output from elf to uf2
git clone https://github.com/rp-rs/pico-blink-rs
cd pico-blink-rs
cargo build --release
../pico-sdk/build/elf2uf2/elf2uf2 target/thumbv6m-none-eabi/release/pico-blink-rs pico-blink-rs.uf2
```

[rp2040_pac](https://github.com/rp-rs/rp2040-pac) seems to have changed since the example was written, so `cargo build` fails with "error[E0609]: no field `gpio25` on type `PADS_BANK0`".  In `main.rs` change:

```diff
     // Configure pin 25 for GPIO
-    p.PADS_BANK0.gpio25.write(|w| {
+    p.PADS_BANK0.gpio[25].write(|w| {
         // Output Disable off
         w.od().clear_bit();
         // Input Enable on
         w.ie().set_bit();
         w
     });
-    p.IO_BANK0.gpio25_ctrl.write(|w| {
+    p.IO_BANK0.gpio[25].gpio_ctrl.write(|w| {
         // Map pin 25 to SIO
-        w.funcsel().sio_25();
+        w.funcsel().sio_0();
         w
     });
```

Finally, flash the uf2 (like with [MicroPython or C/C++](https://www.raspberrypi.org/documentation/rp2040/getting-started/#getting-started-with-c)):

1. Push and hold the __BOOTSEL__ button on the Pico and plug into host USB port
    - Pico should mount as Mass Storage Device "RPI-RPI2" on host
1. Copy UF2 file to "RPI-RP2" volume
1. Pico will reboot and run program

## Coming Soon

[`main.rs`](https://github.com/rp-rs/pico-blink-rs/blob/develop/src/main.rs) is pretty... low-level.  To address this, [rp-hal](https://github.com/rp-rs/rp-hal) is in progress to implement the [Rust Embedded HAL](https://github.com/rust-embedded/embedded-hal).  Once that is complete, the same example might be written like a [similar example](https://github.com/atsamd-rs/atsamd/blob/master/boards/feather_m0/examples/blinky_basic.rs) for the [Adafruit METRO M0](https://www.adafruit.com/product/3505):

```rust
#[entry]
fn main() -> ! {
    let mut peripherals = Peripherals::take().unwrap();
    let core = CorePeripherals::take().unwrap();
    let mut clocks = GenericClockController::with_external_32kosc(
        peripherals.GCLK,
        &mut peripherals.PM,
        &mut peripherals.SYSCTRL,
        &mut peripherals.NVMCTRL,
    );
    let mut pins = hal::Pins::new(peripherals.PORT);
    let mut red_led = pins.d13.into_open_drain_output(&mut pins.port);
    let mut delay = Delay::new(core.SYST, &mut clocks);
    loop {
        delay.delay_ms(200u8);
        red_led.set_high().unwrap();
        delay.delay_ms(200u8);
        red_led.set_low().unwrap();
    }
}
```

Nowhere close to as svelte as the MicroPython version, but then Rust isn't particularly well known for being terse.
