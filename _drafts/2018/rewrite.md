---
layout: post
title: Rewrite
tags:
- netcore
- csharp
- nng
---

[Existing architecture/implementation]({% post_url /2018/2018-10-15-zplus-microservices %}) ("the post-mortem"):

## .NET Core Generic Host

ASP.[]()NET Core also has a ["generic host"](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) for normal applications- those that don't process HTTP requests.

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
```

In `Program.cs`:
```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var host = new HostBuilder()
        .ConfigureHostConfiguration(configBuilder => {
            //...
        })
        .ConfigureServices((hostContext, services) => {
            //...
        })
        .Build();
        await host.RunAsync();
    }
}
```

`ConfigureServices()` is used to register services ("dependencies") with the service ("dependency injection") container.

## Services

[IHostedService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice) defines methods for background tasks that are managed by the host.  [Microsoft.Extensions.Hosting.BackgroundService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice) is a base class for implementing long-running services.

Reading:
- [Background tasks with hosted services](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [Implement background tasks in microservices](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)

Service that starts our [ASP.NET Core WebHost]({% post_url /2018/2018-08-24-aspnetcore-thrift %}):
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

ASP.NET Core uses constructor-based dependency injection (DI) so the constructor parameters (its dependencies) will be resolved from the registered services.

Each `BackgroundService` can be registered with the generic host using [`AddHostedService()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionhostedserviceextensions.addhostedservice) or [`AddSingleton()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionserviceextensions.addsingleton):
```csharp
    // In Program.cs
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
            containers.Add(configuration.CreateContainer());
        }
    }

    public List<T> GetExports<T>()
    {
        var ret = new List<T>();
        foreach (var container in containers)
        {
            ret.AddRange(container.GetExports<T>());
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

Came across this blog that showed how to use json, but didn't really seem complete:
https://dzone.com/articles/read-config-data-in-net-core-test-project-net-core

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host#override-configuration
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/index
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options

[This blog](https://weblog.west-wind.com/posts/2016/May/23/Strongly-Typed-Configuration-Settings-in-ASPNET-Core) shows how to have strongy-typed configuration via `IOptions<T>`.
But [this post](https://www.strathweb.com/2016/09/strongly-typed-configuration-in-asp-net-core-without-ioptionst/) shows how to use POCO configuration without any dependency on `Microsoft.Extensions.Options`.


`Program.cs`:
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
        "nng":{
            "brokerIn": "ipc://zxy-brokerIn",
            "brokerOut": "ipc://zxy-brokerOut"
        },
    },
}
```

Need to copy `appsettings.json` to the output folder.  Add to project file:
```xml
<ItemGroup>
    <None Update="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

each service gets its own configuration section (i.e. `zxy:http`) that is accessible in a type-safe way (via `Config` instance):
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

    protected override Task ExecuteAsync(CancellationToken token)
    {
        var webHostBuilder = WebHost.CreateDefaultBuilder()
        .UseConfiguration(configuration)
        .UseKestrel(options => {
            var port = config.Port;
            options.Listen(IPAddress.Loopback, port);
        })
```

Alternatively, individual values can be loaded with the longer `configuration.GetSection("zxy:http").GetValue<int>("port")`.

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

Definitely read [_Logging in ASP.NET Core_](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/).

```csharp
// Bring extension methods into scope
using Microsoft.Extensions.Logging;
```

```
dotnet <project_name> add package Microsoft.Extensions.Logging.EventSource
```

```csharp
    var host = new HostBuilder()
    .ConfigureLogging((context, loggingBuilder) => {
        loggingBuilder.AddEventSourceLogger();
    })
```

Let's you use [PerfView](https://github.com/Microsoft/perfview).


## Secrets

https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets

## Service

https://www.stevejgordon.co.uk/running-net-core-generic-host-applications-as-a-windows-service


## Nng

Several benefits to replacing [ZeroMQ](http://zeromq.org/) with [NNG](https://github.com/nanomsg/nng)

We've been using [ZeroMQ](http://zeromq.org/) (actually, [NetMQ](https://github.com/zeromq/netmq/)).  Along with [NetMQ.WebSockets](https://github.com/NetMQ/NetMQ.WebSockets) for [WebSocket](https://en.wikipedia.org/wiki/WebSocket) support.

Our overlay and input-hooking (C++) moved to [nanomsg](https://github.com/nanomsg/nanomsg) for IPC/named-pipe support but also supports [WebSocket](https://nanomsg.org/v1.1.4/nn_ws.html).

Did some initial investigation [forking](https://github.com/zplus/NNanomsg) [NNanomsg](https://github.com/mhowlett/NNanomsg) to convert it to .Net Standard and fix some small issues.

In the course of that found [NNG (nanomsg-next-generation)](https://github.com/nanomsg/nng) is being developed as the successor to nanomsg (with the latter now in "maintinence mode").

[forking](https://github.com/zplus/csnng) [csnng](https://github.com/zplus/csnng).  Again, to convert it to .Net Standard and fix some issues.  In the end abandoning it for [nng.NETCore](https://github.com/subor/nng.NETCore).


```csharp
class NngConfig
{
    public string BrokerIn {get; set;}
    public string BrokerOut {get; set;}
}

public class NngBroker : BackgroundService
{
    public NngBroker(FactoryType factory, IConfiguration configuration)
    {
        this.factory = factory;

        // Read configuration
        config = new NngConfig();
        var nngSection = configuration.GetSection("zxy:nng");
        nngSection.Bind(config);
    }

    protected override async Task ExecuteAsync(CancellationToken token)
    {
        // Create simple nng broker that receives messages with puller and publishes them
        var pullSocket = factory.PullerCreate(config.BrokerIn, true).Unwrap();
        var input = pullSocket.CreateAsyncContext(factory).Unwrap();
        var output = factory.PublisherCreate(config.BrokerOut).Unwrap().CreateAsyncContext(factory).Unwrap();

        while (!token.IsCancellationRequested)
        {
            var msg = await input.Receive(token);
            Console.WriteLine("Broker: " + msg);
            if (!await output.Send(msg))
            {
                await Console.Error.WriteLineAsync("Failed!");
            }
        }
    }

    FactoryType factory;
    NngConfig config;
}
```

Where `FactoryType` is dependency for creating nng resources [via NNG native assemly]({% post_url /2018/2018-09-09-native-assembly %}#one-load-context to-rule-them-all) ([nng.NETCore source](https://github.com/subor/nng.NETCore/blob/master/nng.Shared/AssemblyLoadContext.cs)):
```csharp
[Export(typeof(ISingletonPlugin))]
public class NngSingletonPlugin : ISingletonInstancePlugin
{
    public Type ServiceType()
    {
        return typeof(FactoryType);
    }

    public object CreateInstance()
    {
        var path = Path.GetDirectoryName(GetType().Assembly.Location);
        var loadContext = new NngLoadContext(path);
        var factory = NngLoadContext.Init(loadContext);
        return factory;
    }
```

## Conclusion

We've got a good start for a new layer0: microservices loaded from plugins, HTTP server for HTML5 UI, configuration and logging, messaging via Thrift (probably) over NNG, and CI/CD.

Microsoft was hardly the first DI solution and mostly likely isn't the best, but it plays nicely with ASP.NET Core, .NET Core in general, and the rest of the Microsoft ecosystem.