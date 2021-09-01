---
layout: post
title: Tensorflow on the Edge TPU
tags:
- tensorflow
- ai
- rust
- iot
---

Given my fatal addiction to dev boards, I've been contemplating a [device with an Edge TPU](https://coral.ai/products/) (Tensor Processing Unit) for a while.

[CAPA13R](https://rendered-obsolete.github.io/2021/04/13/capa13r.html) has an M.2 slot

https://coral.ai/docs/m2/get-started

Make sure to install Python 3.8 ([issue](https://github.com/google-coral/pycoral/issues/6)).


## Rustified

```
thread 'main' panicked at 'Unable to find libclang: XXX
```

Visual Studio Community
Workloads > Desktop development with C++
Individual components > C++ Clang Compiler for Windows

```powershell
# Note `x64` for 64-bit clang
$env:LIBCLANG_PATH="${env:ProgramFiles(x86)}/Microsoft Visual Studio/2019/Community/VC/Tools/Llvm/x64/bin"
```

```
  C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.29.30037\include\xutility:3799:18: error: 'auto' return without trailing return type; deduced return types are a C++14 extension, err: true
  thread 'main' panicked at 'Unable to generate bindings: ()', C:\XXX\ionosnetworks-tflite-rs\build.rs:256:40
```

```diff
- .clang_arg("-std=c++11")
+ .clang_arg("-std=c++14")
```