---
layout: post
title: Gamepad Input with Rust
tags:
- rust
- input
- libusb
- hidapi
---

Part of our platform includes a user-space device layer that abstracts input to gamepads supporting [DirectInput, XInput](https://docs.microsoft.com/en-us/windows/desktop/xinput/xinput-and-directinput), or [HID](https://docs.microsoft.com/en-us/windows-hardware/drivers/hid/introduction-to-hid-concepts).  It uses [dll injection](https://en.wikipedia.org/wiki/DLL_injection) to attach to games (via [Evolve](https://www.evolvehq.com/)'s tech) and provide input.  It's a fairly large chunk of worrisome C++ with dubious multi-threading that keeps me up at night.

I've been toying with the idea of gradually replacing it with Rust.  To do that I decided to look into accessing HID gamepads from Rust on both Windows and OSX.  Specifically, these devices (cross-reference with [this online list](http://www.linux-usb.org/usb.ids)):

| Gamepad | Vendor Id | Product Id | Notes
|-|-|-|-
| [Microsoft Xbox One](https://www.amazon.com/Xbox-Wireless-Controller-Black-one/dp/B01LPZM7VI/ref=sr_1_4?s=videogames&ie=UTF8&qid=1537161644&sr=1-4&keywords=xbox+one+controller) | 045e | 02d1
| [Ruyi Wireless](http://games.sina.com.cn/wjzx/wjjj/2018-08-03/doc-ihhehtqh3102483.shtml) | 0483 | 5751 | Not yet available
| [Sony PS4](https://www.amazon.com/DualShock-Wireless-Controller-PlayStation-Black-4/dp/B01LWVX2RG/ref=sr_1_3?s=videogames&ie=UTF8&qid=1537161583&sr=1-3&keywords=dualshock%2B4&th=1) | 054c | 09cc
| [Nintendo Switch Pro](https://www.amazon.com/Nintendo-Switch-Pro-Controller/dp/B01NAWKYZ0/ref=sr_1_3?s=videogames&ie=UTF8&qid=1537161618&sr=1-3&keywords=nintendo+switch+pro) | 057e | 2009

Thus far have found two crates that wrap native libraries:
- [libusb](https://github.com/libusb/libusb) via [libusb-rs](https://crates.io/crates/libusb)
- [hidapi](https://github.com/signal11/hidapi) via [hidapi](https://crates.io/crates/hidapi)

## Cargo libusb

Add the following to `Cargo.toml`:
```
[dependencies]
libusb = "0.3.0"
```

Run `cargo build`:
```
process didn't exit successfully: `/Users/jake/projects/input/target/debug/build/libusb-sys-a53f7746ba781f88/build-script-build` (exit code: 101)
--- stderr
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "Failed to run `\"pkg-config\" \"--libs\" \"--cflags\" \"libusb-1.0\"`: No such file or directory (os error 2)"', libcore/result.rs:945:5
```

Oops, no `pkg-config`.  Found [instructions](https://gist.github.com/jl/9e5ebbc9ccf44f3c804e) how to build it on OSX:
```
LDFLAGS="-framework CoreFoundation -framework Carbon" ./configure --with-internal-glib
make
sudo make install
```

Again, `cargo build`:
```
   Compiling libusb-sys v0.2.3
   Compiling libusb v0.3.0
   Compiling input v0.1.0 (file:///Users/jake/projects/input)
    Finished dev [unoptimized + debuginfo] target(s) in 1.33s
```

## libusb Sample

From [github](https://github.com/dcuddeback/libusb-rs) and [docs](https://github.com/dcuddeback/libusb-rs) enumerate USB devices and get identifying strings:
```rust
let timeout = std::time::Duration::from_secs(1);
let mut context = libusb::Context::new().unwrap();
for mut device in context.devices().unwrap().iter() {

    let mut handle = device.open().unwrap();
    let langs = handle.read_languages(timeout);
    if let Ok(langs) = langs {
        for lang in langs.iter() {
            
            let manufacturer = handle.read_manufacturer_string(*lang, &device_desc, timeout).unwrap();
            let product = handle.read_product_string(*lang, &device_desc, timeout).unwrap();
            println!("Lang {:?} manufacturer={} product={}", lang.primary_language(), manufacturer, product);
        }
    }
```

Works for all the devices I had on-hand.

Tried reading values in a loop but it fails with `LIBUSB_ERROR_NOT_FOUND`:
```rust
loop {
    let mut buffer: [u8; 128] = [0; 128];
    let read = handle.read_bulk(libusb_sys::LIBUSB_ENDPOINT_IN, &mut buffer, timeout);
    match read {
        Ok(count) => println!("{}", count),
        Err(error) => println!("{}", error)
    }
    thread::sleep(Duration::from_secs(1));
}
```

Looks like you need to call [`claim_interface()`](http://dcuddeback.github.io/libusb-rs/libusb/struct.DeviceHandle.html#method.claim_interface) before using the device:
```rust
if let Err(error) = handle.claim_interface(0) {
    println!("claim_interface failed: {}", error);
}
```
But on OSX it fails with: `Access denied (insufficient permissions)`.

This seems to be a common problem with libusb on OSX ([libusb FAQ](https://github.com/libusb/libusb/wiki/FAQ#How_can_I_run_libusb_applications_under_Mac_OS_X_if_there_is_already_a_kernel_extension_installed_for_the_device), [github issue](https://github.com/tessel/node-usb/issues/30)).  Tried calling [`detach_kernel_driver()`](file:///Users/jake/projects/input/target/doc/libusb/struct.DeviceHandle.html#method.detach_kernel_driver) but it returns `Operation not supported or unimplemented on this platform`.

Switched over to a Windows 10 machine and then I was back to not having `pkg-config`...

## hidapi

A second option is [hidapi](https://docs.rs/hidapi/0.5.0/hidapi/).  Just by looking at the API it's clearly simplier than libusb.

In `Cargo.toml`:
```
[dependencies]
hidapi = "0.5.0"
```

Simple read loop:
```rust
fn main() {
    let api = hidapi::HidApi::new().unwrap();
    // Print out information about all connected devices
    for device in api.devices() {
        println!("{:?}", device);
    }

    println!("Opening...");
    // Connect to device using its VID and PID (Ruyi controller)
    let (VID, PID) = (0x0483, 0x5751);
    let device = api.open(VID, PID).unwrap();
    let manufacturer = device.get_manufacturer_string().unwrap();
    let product = device.get_product_string().unwrap();
    println!("Product {:?} Device {:?}", manufacturer, product);

    loop {
        println!("Reading...");
        // Read data from device
        let mut buf = [0u8; 8];
        let res = device.read_timeout(&mut buf[..], 1000).unwrap();
        println!("Read: {:?}", &buf[..res]);
        //thread::sleep(Duration::from_secs(1));
    }
}
```

When I first tested this at home with a Nintendo Switch Pro controller, `read_timeout()` always times out and returns no data.  As I was writing this up as a failed/unsucessful experiment and double-checking error codes and so on, I found out it works correctly with our "Ruyi" controller!

For our controller the read returns 10 bytes:
```rust
use std::fmt;
use std::mem;

#[derive(Copy, Clone)]
#[repr(u16)]
enum Buttons {
    DPadLeft    = 0x0001,
    DPadRight   = 0x0002,
    DPadUp      = 0x0004,
    DPadDown    = 0x0008,
    Start       = 0x0010,
    Opt         = 0x0020,
    Home        = 0x0040,
    Share       = 0x0080,
    X           = 0x0400,
    B           = 0x0800,
    Y           = 0x1000,
    A           = 0x2000,
    L1          = 0x4000,
    R1          = 0x8000,
}

#[repr(packed)]
struct RuyiInput {
    header: u16, // First byte is report number?
    buttons: Buttons,
    left_trigger: u8,
    right_trigger: u8,
    left_stick_x: u8,
    left_stick_y: u8,
    right_stick_x: u8,
    right_stick_y: u8,
}

impl RuyiInput {
    fn new(data: [u8; 10]) -> RuyiInput {
        unsafe {
            mem::transmute(data)
        }
    }
}
```

Button presses are bit-flags set in two bytes.  `enum Buttons` represents flags returned by the device, and `#[repr(u16)]` makes it a `u16` (see [this SO](https://stackoverflow.com/questions/25507320/how-to-specify-the-representation-type-for-an-enum-in-rust-to-interface-with-c)).

As per [this SO](https://stackoverflow.com/questions/36061560/can-i-take-a-byte-array-and-deserialize-it-into-a-struct), use `mem::transmute()` to convert the bytes into a `struct RuyiInput`.

I originally had `#[derive(Debug)]` on the `enum` and `struct` and `println!("{:?}", input)` to print it out.  But, on the first loop iteration the program exits with `Illegal instruction: 4`.  Turns out it was because there was no enum value for `0` (or other bit-flag combinations).

Instead, implement `fmt::Display`:
```rust
impl fmt::Display for RuyiInput {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:04x} LT {} RT {} LS x{}y{} RS x{}y{}", self.buttons as u16, 
            self.left_trigger, self.right_trigger, self.left_stick_x, self.left_stick_y, 
            self.right_stick_x, self.right_stick_y)
    }
}
```

[`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html) (and [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html)) are needed on `enum Buttons` otherwise `self.buttons as u16` fails to compile with `cannot move out of borrowed content`.

Next up, sending it somewhere.