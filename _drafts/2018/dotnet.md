---
layout: post
title: A Potpourri of Net Core
tags:
- dotnet
- devops
- productivity
---

I have been using Visual Studio professionally for about a decade now.  [Jenkins](https://jenkins.io/) for CI/CD for the last 5 years.  While the latest iterations of both are excellent tools ([Jenkins pipeline](https://jenkins.io/solutions/pipeline/), in particular), my development environment feels stagnant.

I use OSX at home, and with our [migration to Github]({% post_url /2018/2018-08-11-so-long-to-bitbucket %}) now seemed like the perfect time to try something different.

## dotnet CLI

I'd previously used [MonoDevelop](https://www.monodevelop.com/) for some time.  Post Microsoft acquisition it was rebranded [Visual Studio for Mac](https://docs.microsoft.com/en-us/visualstudio/mac/).  While more comfortable than Apple's Xcode, it always feels... clunky.  [Enter the dotnet CLI](https://docs.microsoft.com/en-us/dotnet/core/tools/?tabs=netcore2x):

```bash
mkdir l0core
cd l0core
# Create l0core.sln
dotnet new sln
# Create l0core/l0core.csproj
dotnet new console --name l0core
# Add l0core.csproj to l0core.sln
dotnet sln add l0core
# Add a bunch of existing projects

dotnet add l0core package <package_name>
```

One thing that surprised me was although you can add an existing project to the sln with:
```
dotnet sln add ../layer0/Layer0Shared/
```
You can't easily add a project reference, for example:
```bash
# NB: This doesn't work
dotnet add l0core reference Layer0Shared
```

Instead, you have to refer to the actual project:
```
dotnet add l0core reference ../layer0/Layer0Shared/
```

Reference `..\..\layer0\Layer0Shared\Layer0Shared.csproj` added to the project.

When I created the project [VS Code C# guide](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code#faq) mentions debugging and stuff that didn't work.  Down at the bottom, the [FAQ](https://docs.microsoft.com/en-us/dotnet/core/tutorials/with-visual-studio-code#faq) mentions `.Net: Generate Assets for Build and Debug`.  Failed with:
```
Could not locate .NET Core project.  Assets were not generated.
```

Seems to be [this bug](https://github.com/OmniSharp/omnisharp-vscode/issues/1425).  In any case, closing and restarting VS Code seemed to fix the issue.

__Start Debugging__ and immediately failing was great:  
![]({{ "/assets/vscode_debug.png" | absolute_url }})

## Visual Studio Code

[Visual Studio Code](https://code.visualstudio.com/) has been my go-to text editor for a while now.  Especially for [editing markdown]({% post_url /2018/2018-02-16-markdown %}).

On 1.26.1 and after 2 weeks using it as my main C# IDE, my gripes are:
- Intellisense stops working frequently (need to __Cmd+Shift+p > `restart omnisharp`__)
- No ["tasks" window](https://docs.microsoft.com/en-us/visualstudio/debugger/using-the-tasks-window?view=vs-2017)
- Starting debugger or runnings tests get stuck.  End up `killall dotnet` and restarting VS Code
- No [XML doc](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/xmldoc/xml-documentation-comments) assistance.  In regular Visual Studio:
  * `///` starts comment block, and hitting return indents and inserts `///`
  * [Tags like `<see>`](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/xmldoc/recommended-tags-for-documentation-comments) auto-complete
- Non-trivial yet common types (e.g. `System.Assembly`) take debugger a while to evaluate


## Appveyor

[Appveyor](https://www.appveyor.com/) can be used for both Windows and Linux builds.

After adding the project via the web interface, either commiting an empty `appveyor.yml` or clicking __NEW BUILD__ will trigger a build and tests.

```
msbuild "C:\projects\nng-netcore\nng.NETCore.sln" /verbosity:minimal /logger:"C:\Program Files\AppVeyor\BuildAgent\Appveyor.MSBuildLogger.dll"
Microsoft (R) Build Engine version 14.0.25420.1
Copyright (C) Microsoft Corporation. All rights reserved.
C:\projects\nng-netcore\nng.NETCore\nng.NETCore.csproj(1,1): error MSB4041: The default XML namespace of the project must be the MSBuild XML namespace. If the project is authored in the MSBuild 2003 format, please add xmlns="http://schemas.microsoft.com/developer/msbuild/2003" to the <Project> element. If the project has been authored in the old 1.0 or 1.2 format, please convert it to MSBuild 2003 format.
```

Appveyor defaults to using msbuild 14.0 which doesn't support the latest project format.  Change my `appveyor.yml` to:
```yml
image: Visual Studio 2017
```

Push (or __NEW BUILD__):
```
C:\Program Files\dotnet\sdk\2.1.401\Sdks\Microsoft.NET.Sdk\targets\Microsoft.PackageDependencyResolution.targets(198,5): error NETSDK1004: Assets file 'C:\projects\nng-netcore\nng.NETCore\obj\project.assets.json' not found. Run a NuGet package restore to generate this file. [C:\projects\nng-netcore\nng.NETCore\nng.NETCore.csproj]
Done Building Project "C:\projects\nng-netcore\nng.NETCore\nng.NETCore.csproj" (default targets) -- FAILED.
```

To `appveyor.yml` add:
```yml
before_build:
  - cmd: dotnet restore
```


## Testing

a [handful](https://andrewlock.net/creating-parameterised-tests-in-xunit-with-inlinedata-classdata-and-memberdata/) [of blog](http://hamidmosalla.com/2017/02/25/xunit-theory-working-with-inlinedata-memberdata-classdata/) [posts](http://ikeptwalking.com/writing-data-driven-tests-using-xunit/)


## Code Coverage

https://www.appveyor.com/blog/2017/03/17/codecov/
https://github.com/codecov/example-csharp
https://github.com/OpenCover/opencover/wiki/Usage

In all projects you want code coverage for:
```xml
<PropertyGroup>
  <DebugType>full</DebugType>
</PropertyGroup>
```

After first build:
`Unable to determine a parent commit to compare against.`

## Deployment

[Branches](https://www.appveyor.com/docs/branches/) shows how to do branch dependent build configuration without creating multiple `appveyor.yml` files.

https://www.appveyor.com/docs/deployment/

Nuget: https://www.appveyor.com/docs/deployment/nuget/
[Nuget]({% post_url /2018/2018-08-18-nuget-sign-upload %}) Taken steps so I can publish my nupkg from a build machine.  Mentions [encryption secure API tokens](https://ci.appveyor.com/tools/encrypt).

[Pushing artifacts to Github releases](https://www.appveyor.com/docs/deployment/github/#configuring-in-appveyoryml)

```yml
deploy:
  provider: NuGet
  api_key:
    secure: OSKjxq8SQmVX8UaVkgaq1aUeGnuXHiTzNZoIi2VR0OMCp/WypCkBY7JbkmoKz497
  artifact: /.*\.nupkg/
  on:
    branch: master
    appveyor_repo_tag: true
```

## Flair

[![NuGet](https://img.shields.io/nuget/v/Subor.nng.NETCore.svg?colorB=brightgreen)](https://www.nuget.org/packages/Subor.nng.NETCore)
```
[![NuGet](https://img.shields.io/nuget/v/PACKAGE.svg?colorB=COLOR)](https://www.nuget.org/packages/PACKAGE)
```

[![Build status](https://img.shields.io/appveyor/tests/jake-ruyi/nng-netcore/master.svg)](https://ci.appveyor.com/project/jake-ruyi/nng-netcore/branch/master)

From the [shields.io examples](https://shields.io/#/examples/build) I got:
```
[![Build status](https://img.shields.io/appveyor/tests/USERNAME/PROJECT/BRANCH.svg)](https://ci.appveyor.com/project/USERNAME/PROJECT/branch/BRANCH)
```

[![codecov](https://codecov.io/gh/subor/nng.NETCore/branch/master/graph/badge.svg)](https://codecov.io/gh/subor/nng.NETCore)

Codecov.io repo __Settings > Badge__
```
[![codecov](https://codecov.io/gh/GITHUB_OWNER/PROJECT/branch/BRANCH/graph/badge.svg)](https://codecov.io/gh/GITHUB_OWNER/PROJECT)
```
