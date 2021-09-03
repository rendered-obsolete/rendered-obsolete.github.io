https://docs.microsoft.com/en-us/windows/win32/winmsg/about-hooks

From https://github.com/microsoft/win32metadata
If you'd like to browse the metadata to see what we're emitting, download [Windows.Win32.winmd](https://github.com/microsoft/win32metadata/raw/master/scripts/BaselineWinmd/Windows.Win32.winmd) and [Windows.Win32.Interop.dll](https://github.com/microsoft/win32metadata/raw/master/scripts/BaselineWinmd/Windows.Win32.Interop.dll) into the same folder and load Windows.Win32.winmd in [ILSpy](https://github.com/icsharpcode/ILSpy/releases/latest).

[Examples](https://github.com/microsoft/windows-samples-rs)

https://github.com/microsoft/windows-docs-rs
```powershell
git clone https://github.com/microsoft/windows-docs-rs
cd windows-docs-rs/crates/bindings/
cargo doc --no-deps -p windows -p bindings --target-dir ..\..\docs
cargo doc --open
```

Add `--open` to cargo command or manually open `windows-docs-rs/docs/doc/bindings/index.html`.

Clone and building docs takes ~10 minutes each.

Open `crates/bindings/build.rs` and comment out offending line:
```rust
error: `Windows.Win32.Media.MediaTransport` not found in metadata
   --> build.rs:397:48
    |
397 |         Windows::Win32::Media::MediaTransport::*,
    |
```

[`IntoParam`](https://github.com/microsoft/windows-rs/blob/master/docs/FAQ.md#how-do-i-read-the-signatures-of-generated-functions-and-methods-whats-with-intoparam)

[Dependency Walker](http://www.dependencywalker.com/)
[Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer)
[Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon)


https://docs.microsoft.com/en-us/windows/win32/dlls/dllmain
https://docs.microsoft.com/en-us/windows/win32/winmsg/using-messages-and-message-queues


Rust inline asm
https://blog.rust-lang.org/inside-rust/2020/06/08/new-inline-asm.html
Need `#![feature(asm)]` and `cargo +nightly build`.


http://jbremer.org/x86-api-hooking-demystified/