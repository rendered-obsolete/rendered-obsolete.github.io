---
layout: post
title: A Potpourri of Net Core
tags:
- csharp
- pinvoke
- osx
- native
- win10
---

Random notes related to netcore that didn't deserve a post of their own.

## Converting libs

```
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(14,12,14,54): error CS0579: Duplicate 'System.Reflection.AssemblyCompanyAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(15,12,15,60): error CS0579: Duplicate 'System.Reflection.AssemblyConfigurationAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(16,12,16,58): error CS0579: Duplicate 'System.Reflection.AssemblyFileVersionAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(18,12,18,54): error CS0579: Duplicate 'System.Reflection.AssemblyProductAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(19,12,19,52): error CS0579: Duplicate 'System.Reflection.AssemblyTitleAttribute' attribute
1>obj\Debug\netstandard2.0\NNanomsg.NETStandard.AssemblyInfo.cs(20,12,20,54): error CS0579: Duplicate 'System.Reflection.AssemblyVersionAttribute' attribute
```

```xml
<PropertyGroup>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
</PropertyGroup>
```

## log4net

We use [log4net](https://logging.apache.org/log4net/) with the following [App.config](https://docs.microsoft.com/en-us/dotnet/framework/configure-apps/):
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
  </configSections>
  <log4net>
    <appender name="LogFileAppender" type="log4net.Appender.FileAppender">
      <param name="File" value="Layer0.log"/>
      <param name="AppendToFile" value="false"/>
      <layout type="log4net.Layout.PatternLayout">
        <param name="Header" value="[Header]\r\n"/>
        <param name="Footer" value="[Footer]\r\n"/>
        <param name="ConversionPattern" value="%d [%-5p] %m%n"/>
      </layout>
    </appender>

    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
      <layout type="log4net.Layout.PatternLayout">
        <param name="Header" value="[Header]\r\n"/>
        <param name="Footer" value="[Footer]\r\n"/>
        <param name="ConversionPattern" value="%d [%-5p] %m%n"/>
      </layout>
    </appender>
    <root>
      <level value="INFO"/>
      <appender-ref ref="ConsoleAppender"/>
      <appender-ref ref="LogFileAppender"/>
    </root>
  </log4net>
...
</configuration>
```

- https://msdn.microsoft.com/en-us/magazine/mt694089.aspx
- https://stackify.com/making-log4net-net-core-work/
- https://dotnetthoughts.net/how-to-use-log4net-with-aspnetcore-for-logging/
    - Which gave rise to the [Microsoft.Extensions.Logging.Log4Net.AspNetCore](https://github.com/huorswords/Microsoft.Extensions.Logging.Log4Net.AspNetCore)
- Stackoverflow [#1](https://stackoverflow.com/questions/46169606/how-to-use-log4net-in-asp-net-core-2-0) and [#2](https://stackoverflow.com/questions/51845450/logging-with-log4net-in-asp-net-core-console-app)


[Configuration in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.1&tabs=basicconfiguration)

```csharp
[assembly: log4net.Config.Repository("Layer0")]
[assembly: log4net.Config.XmlConfigurator(Watch = true)]
namespace Ruyi.Logging
{
    public class Log4NetLogger : IRuyiLogger
    {
        //...
```

As it turns out 
[Documentation](http://logging.apache.org/log4net/release/manual/configuration.html)
[XmlConfiguratorAttribute](http://logging.apache.org/log4net/release/sdk/html/T_log4net_Config_XmlConfiguratorAttribute.htm)

```csharp
[assembly: log4net.Config.XmlConfigurator(Watch = true, ConfigFile = "log4net.config")]
```

However, [layer1]({% post_url /2018/2018-09-04-layer1 %}) (a .net framework exe) needs it's own config file.  So its `App.config` contains the following to override `XmlConfigurator.ConfigFile` with a value of `log4net.layer1.config`:
```xml
<configuration>
  <appSettings>
    <!-- This overrides "ConfigFile" of log4net.Config.XmlConfigurator -->
    <add key="log4net.Config" value="log4net.layer1.config"/>
  </appSettings>
</configuration>
```

## PackageReference

https://blog.nuget.org/20180409/migrate-packages-config-to-package-reference.html
https://docs.microsoft.com/en-us/nuget/reference/migrate-packages-config-to-package-reference
BUG: https://developercommunity.visualstudio.com/content/problem/251740/migrate-packagesconfig-to-packagereference-not-sho.html

## dotnet

I've previously used [MonoDevelop](https://www.monodevelop.com/) for some time.  Post Microsoft acquisition it was rebranded [Visual Studio for Mac](https://docs.microsoft.com/en-us/visualstudio/mac/) and there was much rejoicing.  The honeymoon is over:

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

dotnet add l0core package NDesk.Options
```

One thing that surprised me was although you can add an existing project to the sln with:
```
dotnet sln add ../layer0/Layer0Shared/
```
You couldn't easily add a project reference, for example:
```
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

## VS Code

On 1.26.1 and after 2 weeks using it as my main C# IDE, my gripes are:
- Intellisense stops working frequently (need to __Cmd+Shift+p->`restart omnisharp`__)
- No ["tasks" window](https://docs.microsoft.com/en-us/visualstudio/debugger/using-the-tasks-window?view=vs-2017)
- Starting debugger or runnings tests get stuck.  End up `killall dotnet` and restarting VS Code
- No [XML doc](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/xmldoc/xml-documentation-comments) assistance.  In regular Visual Studio:
  * `///` starts comment block, and hitting return indents and inserts `///`
  * [Tags like `<see>`](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/xmldoc/recommended-tags-for-documentation-comments) auto-complete
- Non-trivial yet common types (e.g. `System.Assembly`) take debugger a while to evaluate


## Appveyor

After adding the project via the web interface either commiting an empty `appveyor.yml` or clicking __NEW BUILD__ will trigger a build and tests.

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
## Code Coverage
https://www.appveyor.com/blog/2017/03/17/codecov/
https://github.com/codecov/example-csharp
https://github.com/OpenCover/opencover/wiki/Usage

After first build:
`Unable to determine a parent commit to compare against.`


## Flair

Codecov.io repo __Settings / Badge__