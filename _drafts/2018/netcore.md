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


Mixing <TargetFramework> and <TargetFrameworks>:
C:\Program Files\dotnet\sdk\3.0.100-preview-009812\Sdks\Microsoft.NET.Sdk\targets\Microsoft.PackageDependencyResolution.targets(220,5): error NETSDK1005: Assets file 'D:\usr\nng.NETCore\tests\obj\project.assets.json' doesn't have a target for '.NETCoreApp,Version=v2.1'. Ensure that restore has run and that you have included 'netcoreapp2.1' in the TargetFrameworks for your project. [D:\usr\nng.NETCore\tests\tests.csproj]


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

## dotnet xunit

https://andrewlock.net/running-tests-with-dotnet-xunit-using-cake/


Add to csproj:
```xml
<ItemGroup>
    <DotNetCliToolReference Include="dotnet-xunit" Version="2.3.1" />
</ItemGroup>
```

`dotnet xunit`

[Make sure](https://github.com/xunit/xunit/issues/1358) run from directory containing project with `DotNetCliToolReference`.

```
dotnet xunit -method "zxy.Tests.HostRests.Test2"
Detecting target frameworks in tests.csproj...
Building for framework netcoreapp2.1...
  zxy -> /Users/jake/projects/zxy/output/Debug/bin/netstandard2.0/zxy.dll
  zxy0 -> /Users/jake/projects/zxy/output/Debug/bin/netcoreapp2.1/zxy0.dll
NngTests.cs(58,26): warning CS0219: The variable 'sendString' is assigned but its value is never used [/Users/jake/projects/zxy/zxy/tests/tests.csproj]
  tests -> /Users/jake/projects/zxy/output/Debug/bin/netcoreapp2.1/tests.dll
Running .NET Core 2.1.0 tests for framework netcoreapp2.1...
It was not possible to find any compatible framework version
The specified framework 'Microsoft.NETCore.App', version '2.1.0' was not found.
  - Check application dependencies and target a framework version installed at:
      /usr/local/share/dotnet/
  - Installing .NET Core prerequisites might help resolve this problem:
      http://go.microsoft.com/fwlink/?LinkID=798306&clcid=0x409
  - The .NET Core framework and SDK can be installed from:
      https://aka.ms/dotnet-download
  - The following versions are installed:
      2.0.0 at [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
      2.0.5 at [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
      2.1.1 at [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
      2.1.2 at [/usr/local/share/dotnet/shared/Microsoft.NETCore.App]
```

## Managed Debug Assistants

https://docs.microsoft.com/en-us/dotnet/framework/debug-trace-profile/diagnosing-errors-with-managed-debugging-assistants

https://docs.microsoft.com/en-us/dotnet/framework/debug-trace-profile/callbackoncollecteddelegate-mda