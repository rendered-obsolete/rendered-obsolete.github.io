---
layout: post
title: .NET Core 3.0 Preview w/ C# 8.0 Nullable Reference Types
tags:
- csharp
- dotnet
- netcore
---

Awoke this morning and saw the [.NET Core 3.0 Preview announcement](https://blogs.msdn.microsoft.com/dotnet/2018/12/04/announcing-net-core-3-preview-1-and-open-sourcing-windows-desktop-frameworks/).

Download available from [github releases](https://github.com/dotnet/core/blob/master/release-notes/3.0/preview/3.0.0-preview1.md).

## C# 8.0

Run `dotnet --list-sdks`:
```
1.1.11 [/usr/local/share/dotnet/sdk]
2.0.0 [/usr/local/share/dotnet/sdk]
2.1.4 [/usr/local/share/dotnet/sdk]
2.1.301 [/usr/local/share/dotnet/sdk]
2.1.302 [/usr/local/share/dotnet/sdk]
3.0.100-preview-009812 [/usr/local/share/dotnet/sdk]
```

To use C# 8, in a project file:
```xml
<PropertyGroup>
    <LangVersion>8.0</LangVersion>
</PropertyGroup>
```

`dotnet build`:
```
/usr/local/share/dotnet/sdk/3.0.100-preview-009812/Sdks/Microsoft.NET.Sdk/targets/Microsoft.PackageDependencyResolution.targets(220,5): error MSB4018: The "ResolvePackageAssets" task failed unexpectedly. [/XXX/zxy/tests/tests.csproj]
/usr/local/share/dotnet/sdk/3.0.100-preview-009812/Sdks/Microsoft.NET.Sdk/targets/Microsoft.PackageDependencyResolution.targets(220,5): error MSB4018: System.IO.InvalidDataException: Found invalid data while decoding. [/XXX/zxy/tests/tests.csproj]
```

`dotnet clean` followed by another `dotnet build` fixed this.

`dotnet test` and everything seems good so far.

## Nullable Reference Types

The feature I've been most interested in is ["nullable reference types"](https://blogs.msdn.microsoft.com/dotnet/2017/11/15/nullable-reference-types-in-csharp/), a compile-time check to avoid dereferencing `null` at run-time.  I've been using the [Nullable Reference Types Preview](https://github.com/dotnet/csharplang/wiki/Nullable-Reference-Types-Preview), and the idea is the compiler generates warnings and you choose to ignore them (_tsk tsk_) or turn them into errors:
```xml
<PropertyGroup>
    <MSBuildTreatWarningsAsErrors>true</MSBuildTreatWarningsAsErrors>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

In the first preview the warnings were on by default, but according to that wiki you now need to opt-in with `[module: System.Runtime.CompilerServices.NonNullTypes]` in each project.  Added that to a project and fail:

```
Program.cs(8,10): error CS8636: Explicit application of 'System.Runtime.CompilerServices.NonNullTypesAttribute' is not allowed. [/XXX/zxy/zxy0/zxy0.csproj]
```

Opened up that assembly and it doesn't even contain `NonNullTypesAttribute`.  Luckily, open-source comes to the rescue because [this issue](https://github.com/dotnet/roslyn/issues/30583) shows what changed:

```csharp
#nullable enable
namespace Test
{
    class Program
    {
        static void Main(string[] args)
        {
            string test = null; // line 8
            System.Console.WriteLine(test.Length);
        }
    }
}
```

__Update__:

Per [this SO](https://stackoverflow.com/questions/53633538/how-to-enable-nullable-reference-types-feature-of-c-sharp-8-0-for-the-whole-proj), can be enabled for an entire project via:
```xml
<PropertyGroup>
  <NullableReferenceTypes>true</NullableReferenceTypes>
  <LangVersion>8.0</LangVersion>
</PropertyGroup>
```

`dotnet build`:
```
Program.cs(10,27): error CS8600: Converting null literal or possible null value to non-nullable type. [/XXX/junk/junk.csproj]
Program.cs(11,31): error CS8602: Possible dereference of a null reference. [/XXX/junk/junk.csproj]
```

No assigning `null` to a type that shouldn't be null.  Nice. Change line 8 to:
```csharp
string? test = null;
```

And it still fails with:
```
Program.cs(9,38): error CS8602: Possible dereference of a null reference. [/Users/jake/projects/junk/junk.csproj]
```

`string?` might be null, but add a check and it compiles:
```csharp
string? test = null;
if (test != null)
    System.Console.WriteLine(test.Length);
```

Unfortunately, this is a compiler heuristic/lint instead of a gurantee provided by the type system like `option` in F#, Rust, etc.  The compiler doesn't fail this:
```csharp
static string test = null;
static void Main(string[] args)
{
    System.Console.WriteLine(test.Length);
}
```

Hopefully they'll keep making improvements.

## Visual Studio

To try this out in VS you need the [Visual Studio 2019 Preview](https://docs.microsoft.com/en-us/visualstudio/releases/2019/release-notes-preview).

Right-click a project and select __Properties > Build__ and:
- In _Treat warnings as errors_ select __All__
- Click __Advanced...__ and for _Language Version_ select __C# 8.0 (beta)__.


![]({{ "/assets/vs_csharp_null_warning.png" | absolute_url }})