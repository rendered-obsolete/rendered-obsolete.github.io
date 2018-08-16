---
layout: post
title: Loading 32/64-bit Native Libraries
tags:
- csharp
- native
---

For several years I've used a well-known trick to selectively load 32/64-bit native libraries in Windows applications:
```csharp
class ADLWrapper
{
    [DllImport("LibADLs")]
    static extern int LibADLs_GetAdapterIndex(IntPtr ptr);

    static ADLWrapper()
    {
        // If 64-bit process, need to load 64-bit native dll.  Otherwise, 32-bit dll.
        // Both dlls need to have same filename and be dllexport'ing the same functions.
        if (System.Environment.Is64BitProcess)
        {
            var handle = LoadLibraryEx(pathTo64bitLibs + "LibADLs.dll", IntPtr.Zero, 0);
            if (handle != IntPtr.Zero)
            {
                //...
            }
        }
        else
        {
            // Load 32-bit dll
        }
    }
}
```

This being slight abuse of [the way dynamic libraries are found](https://docs.microsoft.com/en-us/windows/desktop/dlls/dynamic-link-library-search-order).  In this case a C# wrapper around AMD's Display Library.

Prior to the introduction of [Is64BitProcess](https://docs.microsoft.com/en-us/dotnet/api/system.environment.is64bitprocess?view=netframework-4.7.2) in .Net 4.0, the value of `(IntPtr.Size == 8)` was stashed somewhere.

## GetDelegateForFunctionPointer

[Another approach I first came across](https://github.com/mhowlett/NNanomsg/blob/master/NNanomsg/Interop.cs#L193) when experimenting with [NNanomsg](https://github.com/zplus/NNanomsg) uses [GetDelegateForFunctionPointer()](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.getdelegateforfunctionpointer):
```csharp
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
public delegate int nn_socket_delegate(int domain, int protocol);
public static nn_socket_delegate nn_socket;

static void InitializeDelegates(IntPtr nanomsgLibAddr, NanomsgLibraryLoader.SymbolLookupDelegate lookup)
{
    nn_socket = (nn_socket_delegate)Marshal.GetDelegateForFunctionPointer(lookup(nanomsgLibAddr, "nn_socket"), typeof(nn_socket_delegate));
}
```
Where `lookup` is [`GetProcAddress()`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683212(v=vs.85).aspx) on Windows and `dlsym()` on posix platforms.

A similar approach is [used by gRPC](https://github.com/grpc/grpc/blob/master/src/csharp/Grpc.Core/Internal/UnmanagedLibrary.cs#L114).

Ok, but unwieldly for [large APIs like nng](https://nanomsg.github.io/nng/man/v1.0.0/index.html).

[Advances in nuget packaging]({% post_url /2018/2018-08-15-nupkg-with-native %}) brings a lot of magic, but has the following drawbacks:
- Over-reliance on MSBuild that can be a hassle to debug and get working correctly
- Necessitates setting __Platform target__ to something other than `Any CPU`

