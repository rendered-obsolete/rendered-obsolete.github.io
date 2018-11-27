---
layout: post
title: Nupkg Ergonomics or: The ongoing saga of C# bindings to NNG native library
tags:
- dotnet
- nuget
- csharp
- native
---

The [package we created previously]({% post_url /2018/2018-09-09-native-assembly %}) was painful to use:  
- The nupkg `runtimes/` folder containing platform-specific binaries doesn't get copied somewhere that's obvious.  On installation everything goes into the nuget cache; on OSX `~/.nuget/packages/subor.nng.netcore/`, on Windows `%USERPROFILE%/.nuget/packages/subor.nng.netcore/`
- In Visual Studio, if [__Properties->Build->Platform target__ is set to `x86` or `x64`]({% post_url /2018/2018-08-15-nupkg-with-native %}#consuming-nupkg) the correct shared library will get copied to the output folder

Additionally, we had problems with the nupkg not working because of a disconnect between the development and the nupkg environments.  We want to develop using the same folder structure as what nuget uses.

## Putting the "Run" in Runtime

Adding a [`.props/.target` file](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package#including-msbuild-props-and-targets-in-a-package) is probably unavoidable.  When Nuget installs a package it will `<Import>` files in the package's `build/` folder to the project; `.props` at the top of the project and `.targets` at the bottom.

Our package is [Subor.nng.NETCore](https://www.nuget.org/packages/Subor.nng.NETCore/), so create `build/Subor.nng.NETCore.targets`:
```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup>
    <Content Include="$(MSBuildThisFileDirectory)..\runtimes\**">
      <Link>runtimes\%(RecursiveDir)%(Filename)%(Extension)</Link>
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
</Project>
```

This copies all the `runtimes/` files to the output directory.

To include the `.targets` file in the nupkg, in `nng.NETCore.csproj`:
```xml
<ItemGroup>
  <Content Include="build\**">
    <PackagePath>build\%(RecursiveDir)%(Filename)%(Extension)</PackagePath>
    <Visible>false</Visible>
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

Now, any project that consumes our package will `<Import>` the `.targets` and copy our platform-specific `runtimes/` files to its output folder.

Several online sources mention creating `install.ps1`, but [that was deprecated in Nuget 3.1](https://docs.microsoft.com/en-us/nuget/release-notes/nuget-3.1#deprecated).

## Multi-Targetting

Requiring .NET Standard 2.0 is potentially a steep requirement.  With some [well-placed `#if` blocks](https://github.com/subor/nng.NETCore/commit/d9c9f7a92ee1d32f032bd5df94b3215222cf93a9) it is easy to reduce the requirement to .NET Standard 1.5.  But, this necessitates our package [supports multiple target frameworks](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks).

### The Wrong Way

Created additional projects like `nng.NETCore.15.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard1.5</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="..\nng.NETCore\**\*.cs" Exclude="..\nng.NETCore\obj\**\*.cs" />
  </ItemGroup>
```

The `Exclude` property is to avoid the `AssemblyInfo.cs` that is generated and will result in:
```
obj/Debug/netstandard1.5/nng.NETCore.15.AssemblyInfo.cs(10,12): error CS0579: Duplicate 'System.Reflection.AssemblyCompanyAttribute' attribute [/Users/jake/projects/nng.NETCore/nng.NETCore.15/nng.NETCore.15.csproj]
```

If we had pursued this approach, should probably [move the AssemblyInfo meta-data into the project](https://stackoverflow.com/questions/42138418/equivalent-to-assemblyinfo-in-dotnet-core-csproj).

However, the output assembly `bin/$(Configuration)/netstandard1.5/nng.NETCore.dll` isn't included by `dotnet pack`- it's not a dependency of the main project.

So, we created a project just for packing:
```xml
<PropertyGroup>
    <PackageId>Subor.nng.NETCore</PackageId>
    <!-- SNIP -->
</PropertyGroup>
<ItemGroup>
    <ProjectReference Include="..\nng.Shared\nng.Shared.csproj" />
    <ProjectReference Include="..\nng.Shared.15\nng.Shared.15.csproj" />
</ItemGroup>
```

But this resulted in "multiply-defined" symbols:
```
Bus.cs(9,29): error CS0433: The type 'Defines' exists in both 'nng.Shared.15, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null' and 'nng.Shared, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null' [/XXX/nng.NETCore/nng.NETCore/nng.NETCore.csproj]
```

NB: we later found [something that might fix this](https://blogs.msdn.microsoft.com/kirillosenkov/2015/04/04/how-to-have-a-project-reference-without-referencing-the-actual-binary/).

Time to head to the Internet.  We need a way to include unrelated assemblies in a nupkg.  Turns out this is a re-occuring pain point:
- https://stackoverflow.com/questions/44976879/msbuild-multiple-dll-in-a-single-nuget-package
- https://github.com/NuGet/Home/issues/3891
- https://medium.com/@xavierpenya/building-nuget-packages-with-dotnet-core-943f2fa2f4ca

The available solutions boil down to:
1. Use [TargetsForTfmSpecificBuildOutput](https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#advanced-extension-points-to-create-customized-package) (_yikes_)
1. Use a [nuspec file](https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#packing-using-a-nuspec)
1. One `nupkg` per dependency (the [current recommendation](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package))


Tried to add a nuspec file to what we already had:
```xml
  <PropertyGroup>
    <PackageId>Subor.nng.NETCore</PackageId>
    <!-- SNIP -->
    <RepositoryUrl>https://github.com/subor/nng.NETCore</RepositoryUrl>
    <!-- Added the below -->
    <NuspecFile>nng.NETCore.nuspec</NuspecFile>
  </PropertyGroup>
```

`dotnet pack` yields the mysterious error:
```
/XXX/.nuget/packages/nuget.build.packaging/0.2.2/build/NuGet.Build.Packaging.targets(377,3): error : Path cannot be the empty string or all whitespace. [/XXX/nng.NETCore/nng.NETCore/nng.NETCore.csproj]
```

Apparently you can't mix a nuspec file with the PackageReference format.  In fact, no matter what, we couldn't get `dotnet pack` to use the nuspec with NET Core 2.1.302.  But it doesn't matter because we were barking up the wrong tree anyway.

### The Right Way

.NET has a mechanism to multi-target frameworks: [`<TargetFrameworks>`](https://docs.microsoft.com/en-us/dotnet/standard/frameworks#how-to-specify-target-frameworks) (note the "s").

Change the project to:
```xml
<PropertyGroup>
    <TargetFrameworks>netstandard1.5;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```
And `dotnet pack`... yielded a mysterious error:
```xml
/XXX/nng.NETCore/nng.Shared/nng.Shared.csproj : error MSB4057: The target "_GetBuildOutputFilesWithTfm" does not exist in the project.
```

This had us stumped for a while:  
- Google doesn't turn up anything helpful
- Installed .NET Core 1.1
- `<TargetFramework>netstandard1.5` (no "s") worked
- Tested on a Windows 10 box with Visual Studio 2017, same error
- A new, empty project worked

Bafflement.  Turns out it was self-sabotage.  `<TargetFrameworks>` was broken by:
```xml
<PackageReference Include="NuGet.Build.Packaging" Version="0.2.2" />
```

No idea why that was added, but removing it made `dotnet pack` generate a `nupkg` targetting both frameworks:
```
|____lib
| |____netstandard2.0
| | |____nng.NETCore.dll
| |____netstandard1.5
| | |____nng.NETCore.dll
|____build
| |____Subor.nng.NETCore.targets
|____...
```

Oh... now the project dependency, `nng.Shared.dll`, isn't included.  Apparently `NuGet.Build.Packaging` had been doing it this whole time.

And now we come full circle; need multiple assemblies in a `nupkg` using one of the solutions we outlined above.  Ironically, we inadvertantly rediscovered something I already knew.  Two months ago I read (and commented on) [a post about the same thing](https://dev.to/jamesmh/what-ive-learned-so-far-building-coravel-open-source-net-core-tooling---part-1-1h7k).

## Project Structure

We have two managed assemblies:  

- `nng.NETCore.dll`
- `nng.Shared.dll`

Consumers of the package need a reference to __nng.Shared__ which then loads __nng.NETCore__ at runtime (which subsequently loads the platform specific native library).

We will create two nupkgs:

- [Subor.nng.NETCore.Shared](https://www.nuget.org/packages/Subor.nng.NETCore.Shared/)
  - Contains `lib/netstandard2.0/nng.Shared.dll` (which will be added as a reference)
- [Subor.nng.NETCore](https://www.nuget.org/packages/Subor.nng.NETCore/)
  - References __Subor.nng.NETCore.Shared__
  - `nng.NETCore.dll` in `runtimes/any/lib/netstandard2.0/` (see [docs about TFM, etc.](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks#architecture-specific-folders))

In nng.NETCore.csproj:
```xml
  <PropertyGroup>
    <OutputPath>runtimes\any\lib\</OutputPath>
    <!-- Including assembly as part of runtimes/ so don't want it placed in lib/ -->
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\nng.Shared\nng.Shared.csproj">
      <Private>false</Private>
    </ProjectReference>
  </ItemGroup>

  <!-- Must be run after build so output assembly is in runtimes/ -->
  <Target Name="Runtimes" AfterTargets="Build">
    <ItemGroup>
      <Content Include="runtimes\**">
        <PackagePath>runtimes</PackagePath>
        <Visible>false</Visible>
      </Content>
    </ItemGroup>
  </Target>
</Project>
```

`<OutputPath>` is `runtimes/any/lib/` so our development path is the same as when packaged.  `netstandard2.0/` or `netstandard1.5/` will be automatically appended since `<AppendTargetFrameworkToOutputPath>` and `<AppendRuntimeIdentifierToOutputPath>` are enabled by default.

`<ProjectReference>` to nng.Shared will be turned into a package reference; adding __Subor.nng.NETCore__ package to a project automatically adds __Subor.nng.NETCore.Shared__ as well.  `<Private>false</Private>` ensures `nng.Shared.dll` doesn't get copied to nng.NETCore output; once packaged they won't exist in the same folder.

`<Target Name="Runtimes" AfterTargets="Build">` forces the `<Content>` to get packaged __after__ the build.  Without this `nng.NETCore.dll` won't get copied along with the rest of `runtimes/`.

Nothing special is done in `nng.Shared.csproj`.

## Multi-Platform Multi-Targetting

It's possible to also target .NET Framework:
```xml
<TargetFrameworks>netstandard1.5;netstandard2.0;net461</TargetFrameworks>
```

Building this on OSX fails with:
```
/usr/local/share/dotnet/sdk/2.1.302/Microsoft.Common.CurrentVersion.targets(1179,5): error MSB3644: The reference assemblies for framework ".NETFramework,Version=v4.6.1" were not found. To resolve this, install the SDK or Targeting Pack for this framework version or retarget your application to a version of the framework for which you have the SDK or Targeting Pack installed. Note that assemblies will be resolved from the Global Assembly Cache (GAC) and will be used in place of reference assemblies. Therefore your assembly may not be correctly targeted for the framework you intend. [/Users/jake/projects/nng.NETCore/nng.NETCore/nng.NETCore.csproj]
```

Can use a conditional `PropertyGroup` (from [Cross-platform TargetFrameworks](https://weblog.west-wind.com/posts/2017/Sep/18/Conditional-TargetFrameworks-for-MultiTargeted-NET-SDK-Projects-on-CrossPlatform-Builds)):
```xml
<PropertyGroup Condition=" '$(OS)' == 'Windows_NT' ">
    <TargetFrameworks>netstandard1.5;netstandard2.0;net461</TargetFrameworks>
</PropertyGroup>
<PropertyGroup Condition=" '$(OS)' != 'Windows_NT' ">
    <TargetFrameworks>netstandard1.5;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```

Also need conditional `PackageReference` because [System.Runtime.Loader is not available in .NET Framework](https://github.com/dotnet/corefx/issues/22142):
```xml
<ItemGroup Condition="'$(TargetFramework)' == 'netstandard1.5' or '$(TargetFramework)' == 'netstandard2.0' ">
      <PackageReference Include="system.runtime.loader" Version="4.3.0" />
  </ItemGroup>
```

And then in source files:
```csharp
#if (NETSTANDARD1_5 || NETSTANDARD1_6 || NETSTANDARD2_0)
//...
#endif
```

With [.NET Standard 2.1 right around the corner](https://blogs.msdn.microsoft.com/dotnet/2018/11/05/announcing-net-standard-2-1/) this is becoming unwieldly.

It looks like it could be simplified with [Property Functions](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions).  And eventually came across a decent solution from [this SO](https://stackoverflow.com/questions/25456161/using-conditional-symbol-inside-csproj).  In the project:
```xml
<PropertyGroup Condition="'$(TargetFramework)'=='netstandard1.5' or '$(TargetFramework)'=='netstandard1.6' or '$(TargetFramework)'=='netstandard2.0' or '$(TargetFramework)'=='netstandard2.1'">
    <DefineConstants>NNG_NETSTANDARD1_5_AND_UP</DefineConstants>
</PropertyGroup>

<!-- Conditional PackageReference -->
<ItemGroup Condition="$(DefineConstants.Contains('NNG_NETSTANDARD1_5_AND_UP'))">
    <PackageReference Include="system.runtime.loader" Version="4.3.0" />
</ItemGroup>
```

In C#:
```csharp
#if NNG_NETSTANDARD1_5_AND_UP
//...
#endif
```

Far more manageable.

## Odds and Ends

Miscellaneous nupkg notes that don't really warrant their own posts.

### Testing

[Microsoft docs on adding a local package source](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package#testing-package-installation).

Clearing local nuget cache:
```bash
nuget locals all -clear
```

[This SO](https://stackoverflow.com/a/44463578/10176713) on adding a package source to a project:
```xml
<PropertyGroup>
  <RestoreSources>$(RestoreSources);/XXX/projects/multitargettest;https://api.nuget.org/v3/index.json</RestoreSources>
</PropertyGroup>
```

### When to __Not__ Multi-Target

We've been trying to slim down sln files by moving from direct inclusion of source code projects to nuget packages.  One of these was Apache Thrift.  Thrift contains 3 projects: [2 targetting .NET Framework 3.5 and 4.5](https://github.com/apache/thrift/tree/master/lib/csharp/src), and [one targetting .NET Standard 2.0](https://github.com/apache/thrift/tree/master/lib/netcore/Thrift).

We use .NET Framework 3.5 library in our Unity 3D SDK, and use .NET Standard 2.0 library in the rest of our codebase (which targets either .NET Framework 4.6.1 or .NET Standard 2.0).

We created a `nuspec` to package the 3 unrelated projects without touching the original projects:
```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2012/06/nuspec.xsd">
  <metadata>
    <!-- ... -->
  </metadata>
  <files>
    <file src="thrift\lib\netcore\src\bin\Release\Thrift.dll" target="lib/netstandard2.0/Thrift.dll" />
    <file src="thrift\lib\csharp\src\bin\Release\Thrift.dll" target="lib/net35/Thrift.dll" />
    <file src="thrift\lib\csharp\src\bin\Release\Thrift45.dll" target="lib/net45/Thrift.dll" />
  </files>
</package>
```

But after adding the nuget package to some projects we got compiler errors.  In `obj/project.assets.json`:
```json
"Subor.Thrift/0.11.0.3": {
        ...
        "compile": {
          "lib/net45/Thrift.dll": {}
        },
        "runtime": {
          "lib/net45/Thrift.dll": {}
        }
      },
```

Our .NET Framework 4.6.1 projects opted to use the 4.5 assembly instead of the .NET Standard 2.0 assembly like we wanted.

Out of curiousity, we tried removing the 4.5 assembly from the package.  When faced with the choice between NF 3.5 and NS 2.0, .NET Standard wins out.  But, in the end we decided it was probably best to separate the .NET Framework and [.NET Standard assemblies](https://www.nuget.org/packages/Subor.Thrift.NETCore/) so the end-user can make the choice between NF3.5/NF4.5/NS2.0.

### ContentFiles

While perusing Stack Overflow, came across [this issue about "ContentFiles"](https://stackoverflow.com/questions/51959638/nuget-package-contentfiles-not-copied-to-net-core-project).

Was intrigued because it was an aspect of nupkg we've yet to use.  Found the following helpful resources:
- https://blog.nuget.org/20160126/nuget-contentFiles-demystified.html
- https://github.com/NuGet/Home/issues/2060
- https://github.com/NuGet/Home/issues/1908
- https://docs.microsoft.com/en-us/nuget/reference/nuspec#including-content-files
