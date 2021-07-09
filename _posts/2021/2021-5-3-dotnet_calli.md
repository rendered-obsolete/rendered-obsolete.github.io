---
layout: post
title: Native Code in .NET 5.0 and C# 9.0
tags:
- dotnet
- csharp
- performance
description: Exploring calling native code with function pointers introduced in C# 9.0.
slug: native-code-in-net-5-0-and-c-9-0-1
---

A while back [we covered working with native assemblies](https://rendered-obsolete.github.io/2018/09/09/native-assembly.html).  It's worth re-reading to familiarize yourself with the problem and prior solutions.  There's been a few new options introduced in .NET Core and .NET 5.0.

## NativeLibrary

The [NativeLibrary](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativelibrary) static class was introduced in .NET Core 3.0 and makes it easy to use `DllImport` in a portable way.  To call a function `nng_alloc()` in a native shared library:
```csharp
// In `nng.NETCore` managed assembly
namespace nng.Native.Basic
{
    public sealed class UnsafeNativeMethods
    {
        public const string NngDll = "nng";
        
        [DllImport(NngDll, CallingConvention = CallingConvention.Cdecl)]
        public static extern IntPtr nng_alloc(UIntPtr size);
    }
}

// In assembly that references `nng.NETCore`
namespace benchmarks
{
    class Program
    {
        static void Main(string[] args) 
        {
            NativeLibrary.SetDllImportResolver(typeof(nng.Native.Basic.UnsafeNativeMethods).Assembly, DllImportResolver);
            // Calling DllImport'd method triggers resolver
            nng.Native.Basic.UnsafeNativeMethods.nng_alloc(UIntPtr.Zero);
        }

        static IntPtr DllImportResolver(string libraryName, System.Reflection.Assembly assembly, DllImportSearchPath? searchPath)
        {
            // Load platform-specific unmanaged shared library. e.g.:
            var path = "runtimes/osx-x64/native/libnng.dylib";
            return NativeLibrary.Load(path);
        }
    }
}
```

None of the mess of calling `dlopen()` on Mac/*nix or `LoadLibrary()` on Windows to load a *.so or *.dll, respectively.

If `DYLD_PRINT_LIBRARIES` env variable is set before running the code:
```
dyld: loaded: <264EA187-4189-3CE5-82C3-9746FFE68B66> runtimes/osx-x64/native/libnng.dylib
```

## Function Pointers

.NET 5.0 and C# 9.0 [introduced function pointers](https://devblogs.microsoft.com/dotnet/improvements-in-native-code-interop-in-net-5-0/) as "a performant way to call native functions from C#".

```csharp
public class Pointer
{
    public void CallFunctionPointer()
    {
        // Load platform-specific native library
        var handle = NativeLibrary.Load("xxx");
        // Get address of exported symbol
        var ptr = NativeLibrary.GetExport(handle, "nng_alloc");
        // Function pointers require `unsafe` context
        unsafe
        {
            var nng_alloc = (delegate* unmanaged[Cdecl]<nuint, nint>)ptr;
            // Call through function pointer
            nng_alloc(0);
        }
    }
}
```

[`NativeLibrary.GetExport()`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativelibrary.getexport) returns the address of a symbol exported by a shared library.  It effectively replaces calling the native functions `GetProcAddress()` and `dlsym()` on Windows and Mac/*nix, respectively.

Function pointers use the funky syntax `delegate* unmanaged[Cdecl]<args, retval>` (detailed [here](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/function-pointers)):

| Part | Description
|-|-
| `delegate*` | Reuses [`delegate`](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/) keyword and `*` denotes an unsafe pointer
| `unmanaged` | Either `managed` or `unmanaged` depending on the function
| `[Cdecl]` | Optional after `managed`.  One or more comma-separated [`System.Runtime.CompilerServices.CallConv*`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices) values.  Here [`CallConvCdecl`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.callconvcdecl)
| `<args, retval>` | Function signature types- arguments followed by return value (same as [`System.Func`](https://docs.microsoft.com/en-us/dotnet/api/system.func-2))

## Comparison

A [post on medium](https://medium.com/@jarl.gullberg/an-introduction-to-adl-or-how-to-double-your-native-net-interop-performance-c008e4da54db) compared the performance of different interop approaches a few years ago.  We can re-run the same comparison passing arguments by-value with .NET 5.0 using [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet).

`nng_aio_set_output(null, 9, null)` should be a reasonable choice to benchmark native interop performance because [for values greater than __3__ it immediately returns `EINVAL`](https://github.com/nanomsg/nng/blob/7a0de1b25287f08b73c04d4f9c2834ae265cc382/src/nng.c#L1821).

The results on a 2019 13" MacBook Pro with Quad-core i5 @ 2.4 GHz, 16 GB RAM, running macOS Big Sur (11.3), and .NET 5.0.101:


|                   Method |      Mean |     Error |    StdDev | Ratio | RatioSD |
|------------------------- |----------:|----------:|----------:|------:|--------:|
|             CallDelegate | 16.989 ns | 0.3568 ns | 0.3163 ns | 13.01 |    0.74 |
|            CallDllImport |  7.184 ns | 0.1639 ns | 0.1279 ns |  5.49 |    0.22 |
| CallInterfaceToDllImport | 10.939 ns | 0.3972 ns | 1.1711 ns |  8.07 |    0.75 |
|      CallFunctionPointer |  6.733 ns | 0.1613 ns | 0.2208 ns |  5.22 |    0.30 |
|              CallManaged |  1.305 ns | 0.0548 ns | 0.0563 ns |  1.00 |    0.00 |

Looking at _CallDllImport_ and _CallFunctionPointer_, function pointers are ~5-10% faster than `DllImport`.

_CallDelegate_ uses [Marshal.GetFunctionPointerForDelegate](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.getfunctionpointerfordelegate) to call via a managed `delegate`.  Similar to before, it's slowest by a sizeable margin.

_CallInterfaceToDllImport_ is the way we're currently calling native functions in [nng.NET](https://github.com/jeikabu/nng.NETCore); through an `interface` to static functions decorated with `DllImport`.  Based on these results, leveraging `NativeLibrary` and function pointers we can reduce interop overhead by ~33%.  Not too shabby.

To see what's going on, we can use [ILSpy](https://github.com/icsharpcode/ILSpy) via the [VSCode plugin](https://marketplace.visualstudio.com/items?itemName=icsharpcode.ilspy-vscode) to decompile the assembly.  Open the _Command Palette_ (`CMD+SHIFT+E`), run "ILSpy: Decompile IL Assembly (pick file)", and select the desired assembly/*.dll.  In the _Explorer View_ (`CMD+SHIFT+E`), browse to __ILSPY DECOMPILED MEMBERS > benchmarks > Pointer > CallFunctionPointer__:

```
	IL_0000: ldarg.0
	IL_0001: ldfld method int32 *(native int, uint32, native int) benchmarks.Pointer::nng_aio_set_output /* 04000003 */
	IL_0006: stloc.1
	IL_0007: ldsfld native int [System.Runtime]System.IntPtr::Zero /* 0A000015 */
	IL_000c: ldc.i4.s 9
	IL_000e: ldsfld native int [System.Runtime]System.IntPtr::Zero /* 0A000015 */
	IL_0013: ldloc.1
	IL_0014: calli unmanaged cdecl int32(native int, uint32, native int) /* 11000004 */
	IL_0019: stloc.0
	IL_001a: ret
```

And the decompiled source:
```csharp
public unsafe void CallFunctionPointer()
{
	IntPtr intPtr = nng_aio_set_output;
	int num = ((delegate* cdecl<IntPtr, uint, IntPtr, int>)intPtr)(IntPtr.Zero, 9u, IntPtr.Zero);
}
```

So, for whatever reason it turns our function pointer into an `IntPtr` and then casts it to `cdecl` (which must be an internal symbol because we can't use it directly).  But the key thing is the use of `calli`.  Compare that with __benchmarks > DllImport > CallDllImport__:
```
	IL_0000: ldsfld valuetype [nng.NET.Shared]nng.Native.nng_aio [nng.NET.Shared]nng.Native.nng_aio::Null /* 0A000014 */
	IL_0005: ldc.i4.s 9
	IL_0007: ldsfld native int [System.Runtime]System.IntPtr::Zero /* 0A000015 */
	IL_000c: call int32 [nng.NET]nng.Native.Aio.UnsafeNativeMethods::nng_aio_set_output(valuetype [nng.NET.Shared]nng.Native.nng_aio, uint32, native int) /* 0A00001C */
	IL_0011: stloc.0
	IL_0012: ret
```

`DllImport` uses `call`.  Based on the docs for [call](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.call) and [calli](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.calli) opcodes, `call` uses a "method descriptor" that is a "metadata token" (and works with virtual functions) while `calli` is just "a pointer to an entry point"- and apparently faster.

Additional reading:

- [Understanding System.Runtime.Loader.AssemblyLoadContext](https://docs.microsoft.com/en-us/dotnet/core/dependency-loading/understanding-assemblyloadcontext)
- [Unmanaged (native) library loading algorithm](https://docs.microsoft.com/en-us/dotnet/core/dependency-loading/loading-unmanaged)

