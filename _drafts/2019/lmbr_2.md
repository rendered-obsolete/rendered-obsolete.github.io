---
title: Amazon Lumberyard
published: false
tags: lumberyard,gamedev,rust
canonical_url: 
series: lumberyard
---

## Adding a library to WAF module

https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-using-module.html

For example, in `Sandbox/Editor/wscript`:
```python
# ...
hw = dict(
    # ...
    # OLD: win_lib = ['version'],
    win_lib = ['version', 'staticlib', 'Ws2_32', 'userenv'],
    libpath = ['c:/XXX/lmbr/target/debug/'],
```

If you get "undefined symbol" link errors check out [this issue](https://github.com/rust-lang/rust/issues/52892).

`wrapper.h` must be `wrapper.hpp`

https://forums.developer.apple.com/thread/104296
https://developer.apple.com/documentation/xcode_release_notes/xcode_10_release_notes
```
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```

```sh
grep aux | grep build
# Get PID
sudo env "PATH=$PATH" rust-lldb -p <PID>
```


## Debugging bindgen

The [end of CONTRIBUTING.md](https://github.com/rust-lang/rust-bindgen/blob/master/CONTRIBUTING.md#debug-logging) has some good information.


It's pretty neat, it turns my 3.4MB pre-processed C++ and reduces it to a sub-10-line C++ file.

My first `predicate.sh`:
```sh
#!/usr/bin/env bash

# Exit the script with a nonzero exit code if:
# * any individual command finishes with a nonzero exit code, or
# * we access any undefined variable.
set -eu

~/projects/rust-bindgen/csmith-fuzzing/predicate.py \
    --expect-bindgen-fail \
    --bindgen-args "-- -std=c++14" \
    ./__bindgen.hpp
```

```sh
creduce ./predicate.sh ./__bindgen.hpp
```
reduces it to:
```c++
}
```

Indeed, an invalid source file is the shortest possible way to get bindgen to fail.  I need bindgen to kind of work, but no output bindings:

```sh
~/projects/rust-bindgen/csmith-fuzzing/predicate.py \
    --expect-bindgen-fail \
    --bindgen-args "-- -std=c++14" \
    --bindgen-grep "Unhandled cursor kind 24" \
    ./__bindgen.hpp
```

## Building LLVM

https://clang.llvm.org/get_started.html
Linux VM with 8GB and 1.5GB of swap it ran out of memory linking.

https://llvm.org/docs/CMake.html#llvm-specific-variables
```bash
cmake -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_ENABLE_PROJECTS=clang -Thost=x64 ../llvm
```

```bash
$ LIBCLANG_PATH=<llvm-project>/build/lib cargo build
```
If using Visual Studio and PowerShell:
```powershell
PS> $env:LIBCLANG_PATH="<llvm-project>/build/Debug/bin"
PS> cargo build
```

## Next

```rust
error[E0391]: cycle detected when processing `root::AZ::u32`
  --> /Users/j_woltersdorf/projects/lmbr/target/debug/build/lmbr_sys-4d14e371f0340e58/out/bindings.rs:91:24
   |
91 |         pub type u32 = u32;
   |                        ^^^
   |
   = note: ...which again requires processing `root::AZ::u32`, completing the cycle
note: cycle used when processing `root::AZ::Debug::ProfilerRegister::m_systemId`
  --> /Users/j_woltersdorf/projects/lmbr/target/debug/build/lmbr_sys-4d14e371f0340e58/out/bindings.rs:547:33
   |
547|                 pub m_systemId: root::AZ::u32,
   |                                 ^^^^^^^^^^^^^
```

In `AzCore/base.h`:
```c
namespace AZ
{
#if AZ_TRAIT_COMPILER_INCLUDE_CSTDINT // Defined on Windows/Mac/Linux/Android
    typedef int8_t    s8;
    typedef uint8_t   u8;
    typedef int16_t   s16;
    typedef uint16_t  u16;
    typedef int32_t   s32;
    typedef uint32_t  u32;
#   if AZ_TRAIT_COMPILER_INT64_T_IS_LONG // int64_t is long
    typedef signed long long        s64;
    typedef unsigned long long      u64;
#   else
    typedef int64_t   s64;
    typedef uint64_t  u64;
#   endif
    //...
```
Causes bindgen to generate:
```rust
pub mod AZ {
        //...
        pub type s8 = i8;
        pub type u8 = u8;
        pub type s16 = i16;
        pub type u16 = u16;
        pub type s32 = i32;
        pub type u32 = u32;
        pub type s64 = i64;
        pub type u64 = u64;
```

One possible work-around
```rust
.blacklist_type(r"AZ::u\d{2,3}") // Ban AZ::u32, etc. generated by C typedefs
.raw_line("type U32 = u32;") // Create top-level type aliases
.raw_line("type U64 = u64;")
// Define AZ::u32 in terms of our top-level aliases
.module_raw_lines("root::AZ", ["pub type u32 = crate::U32;", "pub type u64 = crate::U64;"].iter().map(|s| *s))
```


`bindings.rs`:
```rust
/* automatically generated by rust-bindgen */

type U32 = u32; // <<< Our top-level type aliases
type U64 = u64;

#[allow(non_snake_case, non_camel_case_types, non_upper_case_globals)]
pub mod root {

    //...

    #[allow(unused_imports)]
    use self::super::root;
    pub mod AZ {
        #[allow(unused_imports)]
        use self::super::super::root;
        pub type u32 = crate::U32; // << C typedefs defined via top-level aliases
        pub type u64 = crate::U64;
        //...
```

When [primitive types are added to `std`](https://github.com/rust-lang/rust/issues/44865), bindgen could use those and everything would be fine (without the work-around):
```rust
pub mod AZ {
    //...
    pub type u32 = std::primitive::u32;
```

If `LoadLibrary()` returns null and `GetLastError()` is 193 (0xC1 `ERROR_BAD_EXE_FORMAT`) make sure you're building `x64` instead of `x86`.  32-bit applications can't load 64-bit binaries (nor vice-versa).

## C++ Polymorphism

https://hsivonen.fi/modern-cpp-in-rust/
Bindgen and vtables:
https://github.com/rust-lang/rust-bindgen/issues/27

http://jakegoulding.com/rust-ffi-omnibus/objects/

## Gems

```powershell
.\lmbr.exe gems create Rust
```