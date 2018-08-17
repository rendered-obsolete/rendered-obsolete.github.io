---
layout: post
title: Loading 32/64-bit Native Libraries
tags:
- csharp
- native-libs
- win10
---

While possible for C# to call functions in native libraries, you must avoid mixing 32 and 64-bit binaries lest you invite the wrath of `BadImageFormatException`.  Woe unto you.

Different hardware and OSes have different caveats.  For the sake of simplicity, this is talking about 64-bit Windows 10.

## "Any CPU"

[This SO answer](https://stackoverflow.com/questions/5229768/c-sharp-compiling-for-32-64-bit-or-for-any-cpu) covers it nicely.  Basically:
- Binary targetting "Any CPU" will JIT to "any" architecture (x86, x64, ARM, etc.)- but only one at a time.
- On 64-bit Windows, an "Any CPU" binary will JIT to x64, and it can only load x64 native DLLs.

## PInvoke

[Pinvoke](https://docs.microsoft.com/en-us/cpp/dotnet/how-to-call-native-dlls-from-managed-code-using-pinvoke) is one approach to call functions in native DLLs from C#.

For several years I've used a well-known trick to selectively load 32/64-bit native libraries in Windows desktop applications:
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

This being slight abuse of [the way dynamic libraries are found](https://docs.microsoft.com/en-us/windows/desktop/dlls/dynamic-link-library-search-order).  In this case a snippet of C# wrapper around a support library for AMD's Display Library (ADL).

Prior to the introduction of [Is64BitProcess](https://docs.microsoft.com/en-us/dotnet/api/system.environment.is64bitprocess?view=netframework-4.7.2) in .Net 4.0, the value of `(IntPtr.Size == 8)` could be used instead.

When dealing with dynamic libraries with different names (as is the case with ADL), because the argument to [`DllImportAttribute`](https://msdn.microsoft.com/en-us/library/system.runtime.interopservices.dllimportattribute(v=vs.110).aspx) must be a constant we have the unflattering:
```csharp
interface ILibADL
{
    Int64 GetSerialNumber(int adapter);
}

class LibADL64 : ILibADL
{
    [DllImport("atiadlxx.dll")]
    static extern int ADL_Adapter_SerialNumber_Get(int adapter, out Int64 serial);

    public Int64 GetSerialNumber(int adapter)
    {
        Int64 serial;
        if (ADL_Adapter_SerialNumber_Get(adapter, out serial) == 0)
        {
            return serial;
        }
        return -1;
    }
}

class LibADL32 : ILibADL
{
    [DllImport("atiadlxy.dll")]
    static extern int ADL_Adapter_SerialNumber_Get(int adapter, out Int64 serial);

    //...
```

And then somewhere else:
```csharp
if (Environment.Is64BitProcess)
{
    adl = new LibADL64();
}
else
{
    adl = new LibADL32();
}
```

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
Where `lookup` is [`GetProcAddress()`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683212(v=vs.85).aspx) on Windows (and `dlsym()` on posix platforms).

A similar approach is [used by gRPC](https://github.com/grpc/grpc/blob/master/src/csharp/Grpc.Core/Internal/UnmanagedLibrary.cs#L114).

Ok, but unwieldly for [large APIs like nng](https://nanomsg.github.io/nng/man/v1.0.0/index.html) (which is where we're headed).

## SWIG et al.

It's hard to talk about interfacing with native code and not come across [SWIG](http://swig.org/) (or other similar technologies).

Over the last 15 years we've crossed paths a few times.  The first time being when tasked with creating [a python 1.5 wrapper for a Linux virtual device driver](http://www.ittc.ku.edu/kurt/).  Most recently [a half-hearted attempt to use it with AllJoyn]({% post_url /2016/2016-03-02-c-wrapper-for-alljoyn-via-swig %}).

But it always ends the same way: frustration trying to debug my (miss-)use of a non-trivial tool, and befuddlement with the resulting robo-code.

## "Modern" PInvoke

Necessity being the mother of invention, [supporting multiple platforms](https://blog.3d-logic.com/2015/11/10/using-native-libraries-in-asp-net-5/) begat [advances in nuget packaging]({% post_url /2018/2018-08-15-nupkg-with-native %}) to address the issue.  But has the following drawbacks:
- Over-reliance on MSBuild that can be a hassle to debug and get working correctly
- Necessitates setting __Platform target__ to something other than `Any CPU`

## Performance

None of this is "zero-cost".

However, the NNanomsg source code references [this blog](http://ybeernet.blogspot.com/2011/03/techniques-of-calling-unmanaged-code.html) which mentions the [SuppressUnmanagedCodeSecurity](https://docs.microsoft.com/en-us/dotnet/api/system.security.suppressunmanagedcodesecurityattribute) attribute may improve performance significantly.  The security implications aren't immediately clear from the documentation, but it sounds like it may only pertain to operating system resources; it can't be any less safe than calling the library from native code...