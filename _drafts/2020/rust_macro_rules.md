---
layout: post
title: Rust `macro_rules!` Macros
tags:
- rust
- macros
- metaprogramming
series: Rust Macros
---



[Previously]({% post_url /2018/2018-12-03-rust_derive_macros %}), I introduced Rust's declarative and procedural (includes "derive" and "attribute") macros.  And delved into writing a derive mode procedural macro.  This time I'd like to focus on the declarative variety, sometimes called "macros by example" or `macro_rules!` macros.

You've undoubtedly used at least of the the following examples from the standard library:
- [`assert!()` family](https://doc.rust-lang.org/std/macro.assert.html)
- [`panic!()`](https://doc.rust-lang.org/std/macro.panic.html)
- [`println!()` family](https://doc.rust-lang.org/std/macro.println.html)
- [`vec![]`](https://doc.rust-lang.org/std/macro.vec.html)


## Problem

I've been working on [matrixio-rhal](https://github.com/matrix-io/matrix-rhal), a pure Rust re-implementation of the C++ [matrixio_hal_esp32](https://github.com/matrix-io/matrixio_hal_esp32) for the [Matrix Voice with an ESP32]({% post_url /2020/2020-1-6-matrix_voice %}).  It makes use of [bindgen](https://github.com/rust-lang/rust-bindgen) generated FFI bindings to [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/), Espressif's framework for ESP32 SoCs.

Being C, the majority of the ESP-IDF API looks like `int function()`, where the integer return value is `0` for success or another value to indicate what error occurred.

In Rust, I then invariably write a lot of code similar to:
```rust
pub fn setup_event() -> Result<(), Error> {
    unsafe {
        //...
        // Use FFI to call C function
        let retval = esp_idf_sys::gpio_config(&gpio_config);
        // Convert `int` return value to Rust `Result<>` and handle it
        esp_int_into_result(retval)?;
        //...
    }
}
```

Rather than writing this over and over, I'd like to make it more... natural.  Something like `idf!(esp_idf_sys::gpio_config(&gpio_config));`.  Not unlike the [precursor to the `?` operator](https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html), the [`try!` macro](https://doc.rust-lang.org/std/macro.try.html).

## Solution

```rust
#[macro_export]
macro_rules! idf {
    ($x:expr) => {
        {
            let retval = $x;
            esp_int_into_result(retval)?;
        }
    };
}

idf!(esp_idf_sys::gpio_config(&gpio_config));

// BECOMES
{
    let retval = esp_idf_sys::gpio_config(&gpio_config);
    esp_int_into_result(retval)?;
}
```

If the macro is defined in the root of your crate or in another crate, Without `#[macro_export]`, the compiler will complain:
```rust
error: cannot find macro `idf` in this scope
  --> /home/project/src/microphone/esp_mic.rs:31:13
   |
31 |             idf!(esp_idf_sys::gpio_config(&gpio_config));
   |             ^^^
   |
   = help: have you added the `#[macro_use]` on the module/import?
```

Don't let the hint confuse you, `#[macro_use]` won't help.


```rust
pub fn setup_event() -> Result<(), Error> {
    unsafe {
        use esp_idf_sys::*;
        //...
        let retval = gpio_config(&gpio_config);
        esp_int_into_result(retval)?;
        let retval = gpio_install_isr_service(0);
        esp_int_into_result(retval)?;
        let retval = gpio_isr_handler_add(MIC_ARRAY_IRQ, Some(isr_handler), core::ptr::null_mut());
        esp_int_into_result(retval)
    }
}
```

```rust
pub fn setup_event() -> Result<(), Error> {
    unsafe {
        use esp_idf_sys::*;
        //...
        idf! {
            gpio_config(&gpio_config);
            gpio_install_isr_service(0);
            gpio_isr_handler_add(MIC_ARRAY_IRQ, Some(isr_handler), core::ptr::null_mut())
        }
    }
}
```

## Debugging

## Further Information

- ["Declarative Macros" sub-section](https://doc.rust-lang.org/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming) of The Rust Programming Language
- ["Macros By Example" section](https://doc.rust-lang.org/reference/macros-by-example.html) of The Reference
- [The Little Book of Rust Macros](https://danielkeep.github.io/tlborm/book/README.html)