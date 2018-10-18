---
layout: post
title: Rewrite
tags:
- netcore
- csharp
- nng
---

[Existing architecture/implementation]({% post_url /2018/2018-10-15-zplus-microservices %}) ("the post-mortem"):

- [ASP.NetCore]({% post_url /2018/2018-08-24-aspnetcore-thrift %}) there is a ["generic host"](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) for normal applications with 
- Several benefits to replacing [ZeroMQ](http://zeromq.org/) with [NNG](https://github.com/nanomsg/nng)

## .NET Core Generic Host

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var host = new HostBuilder()
        .ConfigureHostConfiguration(configBuilder => {
            configBuilder.AddJsonFile("appsettings.json");
            // configBuilder.AddJsonFile("local.json", optional: true);
            configBuilder.AddCommandLine(args);
        })
        .ConfigureServices((hostContext, services) => {
            services.AddLogging();
            var config = new ServiceConfiguration();
            hostContext.Configuration.Bind("zxy:services", config);
            var zxy = new ZxyContext(services);
            services.AddSingleton<IZxyContext>(zxy);
            PluginLoader.Load(services, config);
        })
        .ConfigureLogging((context, loggingBuilder) => {
            
        })
        .Build();
        await host.RunAsync();
    }
}

```

## Services

Reading:
- [Background tasks with hosted services](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [Implement background tasks in microservices](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)

[ASP.NetCore host]({% post_url /2018/2018-08-24-aspnetcore-thrift %}):
```csharp
public class HttpXy : BackgroundService
{
    public HttpXy(IConfiguration configuration, IZxyContext context)
    {
        //...
    }

    protected override Task ExecuteAsync(CancellationToken token)
    {
        var webHostBuilder = WebHost.CreateDefaultBuilder();
        //...
        webHost = webHostBuilder.Build();
        return webHost.StartAsync(token);
    }

    public override Task StopAsync(CancellationToken token)
    {
        return webHost.StopAsync(token);
    }
}
```

Uses [Microsoft.Extensions.Hosting.BackgroundService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice)

It can be registered with [`AddHostedService()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionhostedserviceextensions.addhostedservice) or [`AddSingleton()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionserviceextensions.addsingleton):
```csharp
    var host = new HostBuilder()
    .ConfigureServices((hostContext, services) => {
        //services.AddSingleton(typeof(IHostedService), typeof(HttpXy));
        services.AddSingleton<IHostedService, HttpXy>();
    })
```

## Plugins

Still using [MEF2/System.Composition]({% post_url /2018/2018-08-16-mef1-mef2 %}).
```csharp
[Export(typeof(IServicePlugin))]
public class HttpPlugin : IServicePlugin
{
    public string Name => "http";
    public Type GetService()
    {
        return typeof(HttpXy);
    }
}
```

[GetExports()](https://docs.microsoft.com/en-us/dotnet/api/system.composition.compositioncontext.getexports) instantiates the classes you [Export](https://docs.microsoft.com/en-us/dotnet/api/system.composition.exportattribute).

```csharp
public class MEF2Plugins
{
    public MEF2Plugins(string path)
    {
        var files = System.IO.Directory.EnumerateFiles(path, "*.dll", System.IO.SearchOption.AllDirectories);
        foreach (var file in files)
        {
            var configuration = new ContainerConfiguration();
            var asm = Assembly.LoadFrom(file);
            configuration.WithAssembly(asm);
            containers.Add(new Plugin { Filename = file, Host = configuration.CreateContainer() });
        }
    }

    public List<T> GetExports<T>()
    {
        var ret = new List<T>();
        foreach (var container in containers)
        {
            ret.AddRange(container.Host.GetExports<T>());
        }
        return ret;
    }
}
```

Add them to DI container:
```csharp
public static void Load(IServiceCollection services, ServiceConfiguration config)
{
    var plugins = mef2.GetExports<IServicePlugin>();
    foreach (var plugin in plugins)
    {
        // Filter enabled plugins
        if (config.IsEnabled(plugin))
        {
            services.AddSingleton(typeof(IHostedService), plugin.GetService());
        }
    }
}
```

[This blog](https://andrewlock.net/using-scrutor-to-automatically-register-your-services-with-the-asp-net-core-di-container/) introduced me to [Scrutor](https://github.com/khellang/Scrutor).


## Configuration

It's perhaps only implied in the post-mortem, but configuration was a bit of a mess.

Came across this blog that showed how to use json, but didn't really seem complete:
https://dzone.com/articles/read-config-data-in-net-core-test-project-net-core

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host#override-configuration
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/index
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options

[This blog](https://weblog.west-wind.com/posts/2016/May/23/Strongly-Typed-Configuration-Settings-in-ASPNET-Core) shows how to have strongy-typed configuration via `IOptions<T>`.
But [this post](https://www.strathweb.com/2016/09/strongly-typed-configuration-in-asp-net-core-without-ioptionst/) shows how to use POCO configuration without any dependency on `Microsoft.Extensions.Options`.


`Main.cs`:
```csharp
using Microsoft.Extensions.Configuration;

var host = new HostBuilder()
.ConfigureHostConfiguration(configBuilder => {
    configBuilder.AddJsonFile("appsettings.json");
    //configBuilder.AddJsonFile("local.json", optional: true);
    configBuilder.AddCommandLine(args);
})
```

[`AddJsonFile(... optional: true)` variant](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.jsonconfigurationextensions.addjsonfile#Microsoft_Extensions_Configuration_JsonConfigurationExtensions_AddJsonFile_Microsoft_Extensions_Configuration_IConfigurationBuilder_System_String_System_Boolean_)

In `appsettings.json`:
```json
{
    "urls": "http://*:8284",
    "zxy":{
        "http":{
            "port":8283
        },
    },
}
```

```csharp
class Config
{
    public int Port {get; set;}
}

public class HttpXy : BackgroundService
{
    IConfiguration configuration;
    Config config;
    IZxyContext zxyContext;

    public HttpXy(IConfiguration configuration, IZxyContext context)
    {
        this.configuration = configuration;
        var section = configuration.GetSection("zxy:http");
        config = new Config();
        section.Bind(config);
        zxyContext = context;
    }
```

In this way each service gets its own configuration section (i.e. `zxy:http`) that is accessible in a type-safe way (via `Config` instance):
```csharp
    protected override Task ExecuteAsync(CancellationToken token)
    {
        var webHostBuilder = WebHost.CreateDefaultBuilder()
        .UseConfiguration(configuration)
        .UseKestrel(options => {
            //var port = configuration.GetSection("zxy:http").GetValue<int>("port");
            var port = config.Port;
            options.Listen(IPAddress.Loopback, port);
        })
```

This works because the default WebHostBuilder [looks for `urls`](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host#server-urls).

This shows setting the listening port using the `zxy:http:port` value I added.

`AddCommandLine()` (in `Main.cs`) allows us to override settings from the command line.  For example, if using Visual Studio Code in `launch.json`:
```json
"configurations": [
    {
        "args": ["--zxy:http:port=9001"],
```

In particular, note that the "section" and "key" values are separated by "__`:`__".

## Logging

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/

## Secrets

https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets

## Service

https://www.stevejgordon.co.uk/running-net-core-generic-host-applications-as-a-windows-service


## Nng

We've been using [ZeroMQ](http://zeromq.org/) (actually, [NetMQ](https://github.com/zeromq/netmq/)).  Along with [NetMQ.WebSockets](https://github.com/NetMQ/NetMQ.WebSockets) for [WebSocket](https://en.wikipedia.org/wiki/WebSocket) support.

Our overlay and input-hooking (C++) moved to [nanomsg](https://github.com/nanomsg/nanomsg) for IPC/named-pipe support but also supports [WebSocket](https://nanomsg.org/v1.1.4/nn_ws.html).

Did some initial investigation [forking](https://github.com/zplus/NNanomsg) [NNanomsg](https://github.com/mhowlett/NNanomsg) to convert it to .Net Standard and fix some small issues.

In the course of that found [NNG (nanomsg-next-generation)](https://github.com/nanomsg/nng) is being developed as the successor to nanomsg (with the latter now in "maintinence mode").

[forking](https://github.com/zplus/csnng) [csnng](https://github.com/zplus/csnng).  Again, to convert it to .Net Standard and fix some issues.  In the end abandoning it for [nng.NETCore](https://github.com/subor/nng.NETCore).
