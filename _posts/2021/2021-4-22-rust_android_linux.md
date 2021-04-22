---
layout: post
title: Rust in Android OS and Linux
tags:
- rust
- news
- linux
- android
---

Google had two recent announcements of particular interest to Rust enthusiasts: ongoing efforts to introduce Rust into [Android OS](https://security.googleblog.com/2021/04/rust-in-android-platform.html) and the [Linux kernel](https://security.googleblog.com/2021/04/rust-in-linux-kernel.html).

While a number of memory-safe languages like Java/Kotlin/go/et al. are used in user-space, the majority of OSes are written in C/C++ where incorrect memory usage is still a source of "high severity" bugs.  The idea is that using Rust would avoid such classes of bugs and provide other benefits.

The [Linux article](https://security.googleblog.com/2021/04/rust-in-linux-kernel.html) has a few specific examples for a simple character device, ioctl handling, and synchronization primitives.  They illustrate some of Rust's advantages: immutability, static typing, lifetime checks, bounds checks, [RAII](https://en.cppreference.com/w/cpp/language/raii), required initialization and error-handling, and so on.  Most achieved at compile-time rather than run-time.

Why not bring C++ to Linux?  [Linus Torvalds had this to say](https://itwire.com/open-source/rust-support-in-linux-may-be-possible-by-5-14-release-torvalds.html):

> LOL.  C++ solves _none_ of the C issues, and only makes things worse. It really is a crap language.

To be fair, C++ _can_ solve some of C's issues, but you generally have to go out of your way and use `unsafe` to anything dubious with Rust.