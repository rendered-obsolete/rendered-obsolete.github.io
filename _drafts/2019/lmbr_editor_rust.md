---
layout: post
title: Lumberyard Editor Plugin in Rust
tags:
- lumberyard
- gamedev
- rust
canonical_url: 
series: Lumberyard Rust
---


```
EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
```

Make sure at least one of the Lumberyard source files contains:
```c
#include <platform_impl.h>
```


If `LoadLibrary()` returns null and `GetLastError()` is `193` (`0xC1`- `ERROR_BAD_EXE_FORMAT`) make sure you're building `x64`, not `x86`.  32-bit applications can't load 64-bit binaries (nor vice-versa).

## C++ Polymorphism

https://hsivonen.fi/modern-cpp-in-rust/
Bindgen and vtables:
https://github.com/rust-lang/rust-bindgen/issues/27

http://jakegoulding.com/rust-ffi-omnibus/objects/


https://github.com/rust-lang/rust/issues/36342
https://github.com/rust-lang/rust/issues/15460