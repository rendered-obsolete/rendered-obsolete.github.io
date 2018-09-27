---
layout: post
title: Gamepad Input with Rust
tags:
- rust
- sdl2
- showdev
---

Continuation of [my previous work]({% post_url /2018/2018-09-17-gamepad-rust %}) handling gamepad input with Rust.

## hidapi DS4

Using [hidapi](https://docs.rs/hidapi/0.5.0/hidapi/) to obtain DualShock 4 (PS4) is also straightforward.

Using the same code as last time but vendor/product id of `0x054c`/`0x09cc` and the following definitions:
```rust
#[derive(Copy, Clone)]
#[repr(u16)]
enum PS4ButtonFlags {
    DPadN       = 0x0000,
    DPadNE      = 0x0001,
    DPadE       = 0x0002,
    DPadSE      = 0x0003,
    DPadS       = 0x0004,
    DPadSW      = 0x0005,
    DPadW       = 0x0006,
    DPadNW      = 0x0007,
    DPadNone    = 0x0008,
    DPadMask    = 0x000F,
    Share       = 0x0010,
    Options     = 0x0020,
    Home        = 0x0040,
    L1          = 0x0100,
    R1          = 0x0200,
    L2          = 0x0400,
    R2          = 0x0800,
    Square      = 0x1000,
    Cross       = 0x2000,
    Circle      = 0x4000,
    Triangle    = 0x8000,
}

struct PS4Input {
    header: u8,
    left_stick_x: u8,
    left_stick_y: u8,
    right_stick_x: u8,
    right_stick_y: u8,
    buttons: PS4ButtonFlags,
    something: u8,
    l2: u8,
    r2: u8,
}
```

The d-pad behaviour is unexpected; when there's no input the lowest [nibble](https://en.wikipedia.org/wiki/Nibble) has a value of `0x8` and a value of `0x0` when you're pressing up.

There's actually many more bytes of input data.  I assume the remainder is related to the gyroscope and touchpad, but didn't investigate.

## SDL

The cross-platform [SDL](https://www.libsdl.org/) library also has Rust bindings: [rust-sdl2](https://github.com/Rust-SDL2/rust-sdl2).

First, install the [SDL 2.0 binaries/framework](http://libsdl.org/download-2.0.php).

To `Cargo.toml` add:
```
[dependencies]
sdl2 = "0.31.0"
sdl2-sys = "0.31.0"
```

On OSX `cargo build` fails with:
```
= note: ld: library not found for -lSDL2
        clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

From [their documentation](https://github.com/Rust-SDL2/rust-sdl2#if-you-are-using-the-sdl2-framework) and [this issue](https://github.com/Rust-SDL2/rust-sdl2/issues/539) need to use:
```
[dependencies]
sdl2-sys = "0.31.0"

[dependencies.sdl2]
features = ["use_mac_framework"]
version = "0.31.0"
```

Minimal program [based off the example in the docs](https://rust-sdl2.github.io/rust-sdl2/sdl2/):
```rust
extern crate sdl2;
extern crate sdl2_sys;

use sdl2::{
    event::Event,
    keyboard::Keycode,
    controller::GameController,
};
use std::collections::HashMap;

fn main() {
    // Initialize SDL
    let sdl_ctx = sdl2::init().unwrap();
    // Initialize game controller subsystem
    let controller_subsystem = sdl_ctx.game_controller().unwrap();
    let mut gamepads: HashMap<u32, GameController> = HashMap::new();
    // Obtain SDL event pump
    let mut event_pump = sdl_ctx.event_pump().unwrap();

    'running: loop {
        // Obtain polling iterator for events
        for event in event_pump.poll_iter() {
            match event {
                Event::Quit {..} |
                Event::KeyDown { keycode: Some(Keycode::Escape), .. } => {
                    break 'running
                },
                Event::KeyDown { keycode: Some(keycode), .. } => {
                    println!("{}", keycode)
                },
                Event::ControllerDeviceAdded { which, ..} => {
                    println!("Device added index={}", which);
                    // When device connected open it so we receive button events
                    let gamepad = controller_subsystem.open(which).unwrap();
                    gamepads.insert(which, gamepad);
                },
                Event::ControllerDeviceRemoved{ which, ..} => {
                    println!("Device removed index={}", which);
                    gamepads.remove(&(which as u32));
                },
                Event::ControllerButtonDown {which, button, ..} => {
                    // Gamepad button pressed
                    println!("Controller index={} button={:?}", which, button);
                },
                _ => {}
            }
        }
    }
}
```

The above works on OSX, but it sounds like other operating systems may require a window in order to receive input events.  In case that's necessary, also investigated creating a window:
```rust
let video_subsystem = sdl_ctx.video().unwrap();

let window = video_subsystem.window("rust-sdl2 demo", 800, 600)
    .position_centered()
    // Window is "hidden" (but may appear in the task-bar)
    .set_window_flags(sdl2_sys::SDL_WindowFlags::SDL_WINDOW_HIDDEN as u32)
    .build()
    .unwrap();
let controller_subsystem = sdl_ctx.game_controller().unwrap();
//controller_subsystem.set_event_state(true); // true by default

// Enable gamepad events when running in background
println!("ALLOW_BACKGROUND_EVENTS={}", sdl2::hint::set("SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS", "1"));
```

The last line is a [hint to SDL](https://wiki.libsdl.org/SDL_HINT_JOYSTICK_ALLOW_BACKGROUND_EVENTS) and seems to be required to obtain input events.  There's a define in `sdl2_sys`:
```rust
pub const SDL_HINT_JOYSTICK_ALLOW_BACKGROUND_EVENTS: &'static [u8; 37] = b"SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS\x00"
```

First attempt, using [`std::str::from_utf8()`](https://doc.rust-lang.org/std/str/fn.from_utf8.html) with a slice of all but the trailing `\0` (which makes `from_utf8()` panic):
```rust
    let bytes = sdl2_sys::SDL_HINT_JOYSTICK_ALLOW_BACKGROUND_EVENTS;
    let hint0 = std::str::from_utf8(&bytes[..bytes.len()-1]).unwrap();
    println!("{}", sdl2::hint::set(hint0, "1"));
```

Goofy, but works.  Alternatively, `std::ffi::CStr` seems to provide the most straight-forward way to work with null-terminated strings:
```rust
let hint1 = std::ffi::CStr::from_bytes_with_nul(bytes).unwrap().to_str().unwrap();
println!("{}", sdl2::hint::set(hint1, "1"));
```

I suspect there's a way to coerce to `CStr` to `&str`, but this works.

## Next

So there's __hidapi__ which [works with our controller]({% post_url /2018/2018-09-17-gamepad-rust %}) and now PS4, but not the Xbox One gamepad.  And then there's __SDL__ which doesn't work with the XBone controller or ours, but works with PS4 (and many other controllers).  SDL also leverages testing and gamepad compatibility work of that project, but at the cost of a pretty big external dependency and _maybe_ headaches related to windows.

Will have to ponder this.