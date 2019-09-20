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


## Gems

https://docs.aws.amazon.com/lumberyard/latest/userguide/gems-system-gems-creating.html
```powershell
.\lmbr.exe gems create Rust
```


https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-builder-custom.html


```
EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryAssertTrace(char const *,...)" (?CryAssertTrace@@YAXPEBDZZ)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "bool __cdecl CryAssert(char const *,char const *,unsigned int,bool *)" (?CryAssert@@YA_NPEBD0IPEA_N@Z)
          EditorRustPlugin.lib(rust_plugin_main.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
          EditorRustPlugin.lib(EditorRustPlugin.cpp.5844958.obj) : error LNK2001: unresolved external symbol "void __cdecl CryDebugBreak(void)" (?CryDebugBreak@@YAXXZ)
```

Make sure at least one of your source files contains:
```c
#include <platform_impl.h>
```