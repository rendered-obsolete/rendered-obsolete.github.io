---
layout: post
title: Criterion
tags:
- rust
- git
---


[gnuplot](http://www.gnuplot.info/)

[3rd-party OSX binaries](https://csml-wiki.northwestern.edu/index.php/Binary_versions_of_Gnuplot_for_OS_X)

```
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:355:21
note: Run with `RUST_BACKTRACE=1` for a backtrace.
error: bench failed
```

```
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', libcore/option.rs:355:21
stack backtrace:
   0: std::sys::unix::backtrace::tracing::imp::unwind_backtrace
   1: std::sys_common::backtrace::print
   2: std::panicking::default_hook::{{closure}}
   3: std::panicking::default_hook
   4: std::panicking::rust_panic_with_hook
   5: std::panicking::continue_panic_fmt
   6: rust_begin_unwind
   7: core::panicking::panic_fmt
   8: core::panicking::panic
   9: criterion_plot::version
  10: <criterion::Criterion as core::default::Default>::default
  11: benchmarks::main
  12: std::rt::lang_start::{{closure}}
  13: std::panicking::try::do_call
  14: __rust_maybe_catch_panic
  15: std::rt::lang_start_internal
  16: main
error: bench failed
```

[XQuartz](https://www.xquartz.org/)