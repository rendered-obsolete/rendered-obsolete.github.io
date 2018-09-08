---
layout: post
title: Loading 32/64-bit Native Libraries in C#
tags:
- csharp
- pinvoke
- osx
- native
- win10
---

While possible for C# to call functions in native libraries, you must avoid mixing 32 and 64-bit binaries lest you invite the wrath of `BadImageFormatException`.  Woe unto you.

Different hardware and OSes have different caveats.  This is mostly talking about 64-bit Windows 10 and occasionally OSX (64 bit).

## "Any CPU"

.Net assemblies can be compiled with a platform target of "Any CPU".  [This SO answer](https://stackoverflow.com/questions/5229768/c-sharp-compiling-for-32-64-bit-or-for-any-cpu) covers it nicely:
- Binary targetting "Any CPU" will [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation) to "any" architecture (x86, x64, ARM, etc.)- but only one at a time.
- On 64-bit Windows, an "Any CPU" binary will JIT to x64, and it can only load x64 native DLLs.

What happens if you try to load a 32-bit assembly into a 64-bit process (or vice-versa)?  [`BadImageFormatException`](https://docs.microsoft.com/en-us/dotnet/api/system.badimageformatexception?view=netframework-4.7.2).

## Windows "Hack"

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

This assumes you have two identically named shared libraries with different paths.  For example, `x86/libadls.dll` and `x64/libadls.dll`.  It's slight abuse of [the way dynamic libraries are found](https://docs.microsoft.com/en-us/windows/desktop/dlls/dynamic-link-library-search-order).  In this case a snippet of C# wrapper around a support library for AMD's Display Library (ADL).

Prior to the introduction of [Is64BitProcess](https://docs.microsoft.com/en-us/dotnet/api/system.environment.is64bitprocess?view=netframework-4.7.2) in .Net 4.0, the value of `(IntPtr.Size == 8)` could be used instead.

When dealing with dynamic libraries with different names (as is the case with ADL), because the argument to [`DllImportAttribute`](https://msdn.microsoft.com/en-us/library/system.runtime.interopservices.dllimportattribute(v=vs.110).aspx) must be a constant we have the unflattering:
```csharp
interface ILibADL
{
    Int64 GetSerialNumber(int adapter);
}

class LibADL64 : ILibADL
{
    [DllImport("atiadlxx.dll")] // 64-bit dll
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
    [DllImport("atiadlxy.dll")] // 32-bit dll
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

Over the last 15 years we've crossed paths a few times.  The first time being when tasked with creating [a python 1.5 wrapper for a Linux virtual device driver](http://www.ittc.ku.edu/kurt/).  Most recently, [a half-hearted attempt to use it with AllJoyn]({% post_url /2016/2016-03-02-c-wrapper-for-alljoyn-via-swig %}).

But it always ends the same way: frustration trying to debug my (miss-)use of a non-trivial tool, and befuddlement with the resulting robo-code.

## "Modern" Nupkg

Necessity being the mother of invention, [supporting multiple platforms](https://blog.3d-logic.com/2015/11/10/using-native-libraries-in-asp-net-5/) begat [advances in nuget packaging]({% post_url /2018/2018-08-15-nupkg-with-native %}) to address the issue.

We've been trying to get a [csnng](https://github.com/zplus/csnng) nupkg containing a native library ([nng](https://github.com/nanomsg/nng)) working on OSX:
1. Build the dynamic library (libnng.dylib):
    ```
    mkdir build && cd build
    cmake -G Ninja -DBUILD_SHARED_LIBS=ON ..
    ```
1. Copy into `runtimes/osx-x64/native` of the nupkg

Set following debug environment variables:
```
# .NET Framework
MONO_LOG_MASK=dll
MONO_LOG_LEVEL=info
# .NET Core
DYLD_PRINT_LIBRARIES=YES
```
- Visual Studio for Mac: Right-click project Options->Run.Configurations.Default under "Environment Variables"
- Visual Studio Code: in `"configurations"` section of `.vscode/launch.json` add `"env": { "DYLD_PRINT_LIBRARIES":"YES" }`

In [RepSocket.cs](https://github.com/zplus/csnng/blob/master/src/Nanomsg2.Sharp/Protocols/Reqrep/RepSocket.cs#L23):
```csharp
[DllImport("nng", EntryPoint = "nng_rep0_open", CallingConvention = Cdecl)]
[return: MarshalAs(I4)]
private static extern int __Open(ref uint sid);
```

Which works fine on Windows, but on OSX:
```
Mono: DllImport error loading library 'libnng': 'dlopen(libnng, 9): image not found'.
Mono: DllImport error loading library 'libnng.dylib': 'dlopen(libnng.dylib, 9): image not found'.
Mono: DllImport error loading library 'libnng.so': 'dlopen(libnng.so, 9): image not found'.
Mono: DllImport error loading library 'libnng.bundle': 'dlopen(libnng.bundle, 9): image not found'.
Mono: DllImport error loading library 'nng': 'dlopen(nng, 9): image not found'.
Mono: DllImport error loading library '/Users/jake/test/bin/Debug/libnng': 'dlopen(/Users/jake/test/bin/Debug/libnng, 9): image not found'.
Mono: DllImport error loading library '/Users/jake/test/bin/Debug/libnng.dylib': 'dlopen(/Users/jake/test/bin/Debug/libnng.dylib, 9): image not found'.
Mono: DllImport error loading library '/Users/jake/test/bin/Debug/libnng.so': 'dlopen(/Users/jake/test/bin/Debug/libnng.so, 9): image not found'.
...
Mono: DllImport error loading library '/Library/Frameworks/Mono.framework/Versions/5.12.0/lib/libnng.dylib': 'dlopen(/Library/Frameworks/Mono.framework/Versions/5.12.0/lib/libnng.dylib, 9): image not found'.
...
```

Solution comes from [Using Native Libraries in ASP.NET 5](https://blog.3d-logic.com/2015/11/10/using-native-libraries-in-asp-net-5/) blog:
1. Preload the dylib (similar to Windows)
1. Use `DllImport("__Internal")`

Code initially based off [Nnanomsg](https://github.com/mhowlett/NNanomsg):
```csharp
static LibraryLoader()
{
    if (Environment.OSVersion.Platform == PlatformID.Unix ||
                Environment.OSVersion.Platform == PlatformID.MacOSX ||
                // Legacy mono value.  See https://www.mono-project.com/docs/faq/technical/
                (int)Environment.OSVersion.Platform == 128)
    {
        LoadPosixLibrary();
    }
    else
    {
        LoadWindowsLibrary();
    }
}

static void LoadPosixLibrary()
{
    const int RTLD_NOW = 2;
    string rootDirectory = AppDomain.CurrentDomain.BaseDirectory;
    string assemblyDirectory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);

    // Environment.OSVersion.Platform returns "Unix" for Unix or OSX, so use RuntimeInformation here
    var isOsx = System.Runtime.InteropServices.RuntimeInformation.IsOSPlatform(OSPlatform.OSX);
    string libFile = isOsx ? "libnng.dylib" : "libnng.so";
    // x86 variants aren't in https://docs.microsoft.com/en-us/dotnet/core/rid-catalog
    string arch = (isOsx ? "osx" : "linux") + "-" + (Environment.Is64BitProcess ? "x64" : "x86");

    var paths = new[]
        {
            Path.Combine(rootDirectory, "runtimes", arch, "native", libFile),
            Path.Combine(rootDirectory, libFile),
            Path.Combine("/usr/local/lib", libFile),
            Path.Combine("/usr/lib", libFile)
        };

    foreach (var path in paths)
    {
        if (path == null)
        {
            continue;
        }

        if (File.Exists(path))
        {
            var addr = dlopen(path, RTLD_NOW);
            if (addr == IntPtr.Zero)
            {
                // Not using NanosmgException because it depends on nn_errno.
                var error = Marshal.PtrToStringAnsi(dlerror());
                throw new Exception("dlopen failed: " + path + " : " + error);
            }
            NativeLibraryPath = path;
            return;
        }
    }

    throw new Exception("dlopen failed: unable to locate library " + libFile + ". Searched: " + paths.Aggregate((a, b) => a + "; " + b));
}

[DllImport("libdl")]
static extern IntPtr dlopen(String fileName, int flags);

[DllImport("libdl")]
static extern IntPtr dlerror();

[DllImport("libdl")]
static extern IntPtr dlsym(IntPtr handle, String symbol);
```

The use of [System.Runtime.InteropServices.RuntimeInformation](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.runtimeinformation) came from [this blog](https://json-formatter.codepedia.info/dotnet-core-to-detect-operating-system-os-platform/).

We construct a path based on the host OS (Linux vs OSX, 32 vs 64-bit), and pass it to `dlopen()` to pre-load the shared library.

Note the absence of file extension with `[DllImport("libdl")]`.  This will load `libdl.dylib` on OSX and `libdl.so` on Linux.

Change the imported function to:
```csharp
[DllImport("__Internal", EntryPoint = "nng_rep0_open", CallingConvention = Cdecl)]
[return: MarshalAs(I4)]
private static extern int __Open(ref uint sid);
```

Debug output:
```
Native library: /Users/jake/test/bin/Debug/runtimes/osx-x64/native/libnng.dylib
Mono: DllImport attempting to load: '__Internal'.
Mono: DllImport loaded library '(null)'.
Mono: DllImport searching in: '__Internal' ('(null)').
Mono: Searching for 'nng_rep0_open'.
Mono: Probing 'nng_rep0_open'.
Mono: Found as 'nng_rep0_open'.
```

Works, but requiring every `DllImport` use `"__Internal"` leaves a lot to be desired.  There's a few alternatives:
- Use [T4 template](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates?view=vs-2017) (or other script) to juggle the pinvoke imports needed by each platform
- Setting [DYLD_xxx_PATH](https://www.mono-project.com/docs/advanced/pinvoke/#macos-framework-and-dylib-search-path)
- [`.config` file with `<dllmap>`](https://www.mono-project.com/docs/advanced/pinvoke/#library-names)
- Just copying the dylib to the output path as part of the build (arbitrary scripts can be included as part of a nupkg)

But has the following drawbacks:
- Over-reliance on MSBuild that can be a hassle to debug and get working correctly
- Necessitates setting __Platform target__ to something other than `Any CPU`

## One Load Context to Rule Them All

Came across ["Best Practices for Assembly Loading"](https://docs.microsoft.com/en-us/dotnet/framework/deployment/best-practices-for-assembly-loading).

Starting looking for information on these "load contexts" and found [an interesting document](https://github.com/dotnet/coreclr/blob/master/Documentation/design-docs/assemblyloadcontext.md#pinvoke-resolution):

> Custom LoadContext can override the AssemblyLoadContext.LoadUnmanagedDll method to intercept PInvokes from within the LoadContext instance so that can be resolved from custom binaries

Ho ho.

Also came across [this post](https://natemcmaster.com/blog/2018/07/25/netcore-plugins/) of someone that works on ASP.[]()NET Core.  [He's using `AssemblyLoadContext` to wrangle plugins](https://github.com/natemcmaster/DotNetCorePlugins), but mentions `LoadUnmanagedDll` is ["the only good way to load unmanaged binaries dynamically"](https://natemcmaster.com/blog/2018/07/25/netcore-plugins/#assemblyloadcontext-the-dark-horse).

To get started, need [System.Runtime.Loader](https://www.nuget.org/packages/System.Runtime.Loader/): `dotnet add nng.NETCore package system.runtime.loader`

First attempt hard-coding paths and filenames:
```csharp
public class ALC : System.Runtime.Loader.AssemblyLoadContext
{
    protected override Assembly Load(AssemblyName assemblyName)
    {
        if (assemblyName.Name == "nng.NETCore")
            return LoadFromAssemblyPath("/Users/jake/nng.NETCore/bin/Debug/netstandard2.0/nng.NETCore.dll");
        // Return null to fallback on default load context
        return null;
    }
    protected override IntPtr LoadUnmanagedDll(string unmanagedDllName)
    {
        // Native nng shared library
        return LoadUnmanagedDllFromPath("/Users/jake/nng/build/libnng.dylib");
    }
}
```

[`DllImport`ed methods must be `static`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/extern) so can't use [`Activator.CreateInstance()`](https://docs.microsoft.com/en-us/dotnet/api/system.activator.createinstance?view=netframework-4.7.2) to easily get at them.  Could probably use [reflection](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection) to extract them all, but that would be unwieldy.

I think the key is from [that LoadContext design doc](https://github.com/dotnet/coreclr/blob/master/Documentation/design-docs/assemblyloadcontext.md#how-load-is-attempted):

> If an assembly A1 triggers the load of an assembly C1, the latter's load is attempted within the LoadContext instance of the former

Basically, once a load context loads an assembly, subsequent dependent loads go through the same context.  So, I moved a factory I use to create objects for my tests into the assembly with the pinvoke methods:

```csharp
TestFactory factory;

[Fact]
public async Task PushPull()
{
    var alc = new ALC();
    var assem = alc.LoadFromAssemblyName(new System.Reflection.AssemblyName("nng.NETCore"));
    var type = assem.GetType("nng.Tests.TestFactory");
    factory = (TestFactory)Activator.CreateInstance(type);
    //...
```

Can't easily call the `static` pinvoke methods directly, but we can use a custom load context to instantiate a type which then calls the pinvokes.

I rarely find exceptions exciting, but this one is:
```
Exception thrown: 'System.InvalidCastException' in tests.dll: '[A]nng.Tests.TestFactory cannot be cast to [B]nng.Tests.TestFactory. Type A originates from 'nng.NETCore, Version=0.0.1.0, Culture=neutral, PublicKeyToken=null' in the context 'Default' at location '/Users/jake/nng.NETCore/tests/bin/Debug/netcoreapp2.1/nng.NETCore.dll'. Type B originates from 'nng.NETCore, Version=0.0.1.0, Culture=neutral, PublicKeyToken=null' in the context 'Default' at location '/Users/jake/nng.NETCore/tests/bin/Debug/netcoreapp2.1/nng.NETCore.dll'.'
```

Different load contexts, different types.

I'm referencing `nng.NETCore` assembly in my test project and also trying to load it here.  How am I supposed to use a type I don't know about?  This is an opportunity for a C# feature I never use, [`dynamic`](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/using-type-dynamic):

```csharp
dynamic factory = Activator.CreateInstance(type);
//...
var pushSocket = factory.CreatePusher(url, true);
```

Test passes, hit breakpoints most of the places I expect (neither VS Code nor VS for Mac can hit breakpoints through `dynamic`), but if I set `DYLD_PRINT_LIBRARIES` my assemblies are conspiculously absent:
```
dyld: loaded: /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.1.2/System.Globalization.Native.dylib
dyld: loaded: /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.1.2/System.Native.dylib
Microsoft (R) Test Execution Command Line Tool Version 15.7.0
Copyright (c) Microsoft Corporation.  All rights reserved.

dyld: loaded: /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.1.2/System.Security.Cryptography.Native.Apple.dylib
Starting test execution, please wait...

Total tests: 1. Passed: 1. Failed: 0. Skipped: 0.
Test Run Successful.
Test execution time: 1.3259 Seconds
dyld: unloaded: /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.1.2/libhostpolicy.dylib
dyld: unloaded: /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.1.2/libhostpolicy.dylib
```

It would seem `AssemblyLoadContext.LoadFrom*()` doesn't use `dyld`?  Hmm... not sure about that.

Obviously, I don't want to use `dynamic` all over the place.  I refactored things; remove the test assembly reference to the pinvoke assembly, and introduce a "middle-man"/glue assembly containing interfaces both use:

| Assembly | Project References | Dynamically Loads | Notes
|-|-|-|-
| "tests" | interfaces | pinvoke | Unit/integration tests
| "interfaces" | | | `interface`s of high-level types using P/Invoke
| "pinvoke" | interfaces | | P/Invoke methods and wrapper types that use them

That enables me to write the very sensible:
```csharp
[Fact]
public async Task PushPull()
{
    var alc = new ALC();
    var assem = alc.LoadFromAssemblyName(new System.Reflection.AssemblyName("nng.NETCore"));
    var type = assem.GetType("nng.Tests.TestFactory");
    IFactory<NngMessage> factory = (IFactory<NngMessage>)Activator.CreateInstance(type);
    var pushSocket = factory.CreatePusher("ipc://test", true);
```

And now I can load native binaries from anywhere I like.

## Performance

None of this is "zero-cost".

The NNanomsg source code references [this blog](http://ybeernet.blogspot.com/2011/03/techniques-of-calling-unmanaged-code.html) which mentions the [SuppressUnmanagedCodeSecurity](https://docs.microsoft.com/en-us/dotnet/api/system.security.suppressunmanagedcodesecurityattribute) attribute may improve performance significantly.  The security implications aren't immediately clear from the documentation, but it sounds like it may only pertain to operating system resources; it can't be any less safe than calling the library from native code...

There's [numerous methods to manipulate messages](https://nanomsg.github.io/nng/man/v1.0.0/nng_msg.5.html) and I'll write them in pure C# to avoid doing pinvoke for trivial string operations.