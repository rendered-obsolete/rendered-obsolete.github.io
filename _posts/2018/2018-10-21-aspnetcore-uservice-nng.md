---
layout: post
title: Microservice-based Application with ASP.NET Core Generic Host
tags:
- netcore
- csharp
- nng
- microservices
---

The [current implementation of our "Z+" platform]({% post_url /2018/2018-10-15-zplus-microservices %}) is basically fine, but we could definitely make improvements.  We've started prototyping a "second generation" client based on ASP.[]()NET Core.

## .NET Core Generic Host

Most coverage of [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) focuses on the [Web Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host) for hosting web apps via the (very excellent) [Kestral web server](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel).  However, it also has a ["generic host"](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host) for normal applications- those that don't process HTTP requests.

Add `Microsoft.Extensions.Hosting` package to your project:
```
dotnet <project_name> add package Microsoft.Extensions.Hosting
```

Bring extension methods and types into scope:
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

On its own this doesn't do much.  But `ConfigureServices()` is used to register services ("dependencies") with the service ("dependency injection") container.

## Services

[IHostedService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.ihostedservice) defines methods for background tasks that are managed by the host.  [Microsoft.Extensions.Hosting.BackgroundService](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice) is a base class for implementing long-running services.  Reading:
- [Background tasks with hosted services](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [Implement background tasks in microservices](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)

Here's a service that starts our [ASP.NET Core WebHost]({% post_url /2018/2018-08-24-aspnetcore-thrift %}):
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

ASP.[]()NET Core uses constructor-based dependency injection (DI) so the constructor parameters (its dependencies) will be resolved from the registered services.

Each `BackgroundService` can be registered with the generic host using [`AddHostedService()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionhostedserviceextensions.addhostedservice) or [`AddSingleton()`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionserviceextensions.addsingleton):
```csharp
    // In Program.cs
    var host = new HostBuilder()
    .ConfigureServices((hostContext, services) => {
        //services.AddSingleton(typeof(IHostedService), typeof(HttpXy));
        services.AddSingleton<IHostedService, HttpXy>();
    })
```

Now their startup and lifetime are managed by the host.

## Plugins

We're still using [MEF2/System.Composition]({% post_url /2018/2018-08-16-mef1-mef2 %}) to break our program into loadable shared libraries.  The assembly for our HTTP service also contains:
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

This time we're keeping the classes we [Export](https://docs.microsoft.com/en-us/dotnet/api/system.composition.exportattribute) as simple as possible because
 [GetExports()](https://docs.microsoft.com/en-us/dotnet/api/system.composition.compositioncontext.getexports) instantiates them.  Previously, some of our types had non-trivial initialization code, such that they were expensive to load even if we ended up not using them.

We create composition containers for all shared libraries found in our plugin directory:
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

Add all services found to the DI container:
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

[This blog](https://andrewlock.net/using-scrutor-to-automatically-register-your-services-with-the-asp-net-core-di-container/) introduced us to [Scrutor](https://github.com/khellang/Scrutor) which looks like an alternative to using MEF tailored to ASP.[]()NET Core.  We might look into it later.


## Configuration

ASP.[]()NET Core has an extensive system for application configuration.  Background reading:
- Early on came across [this blog](https://dzone.com/articles/read-config-data-in-net-core-test-project-net-core) on how to use JSON, but doesn't really seem complete
- [Here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/index) is the MS documentation on configuration in ASP.[]()NET Core
- Closely related is [safe-guarding sensitive data](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) like passwords
- [This blog](https://weblog.west-wind.com/posts/2016/May/23/Strongly-Typed-Configuration-Settings-in-ASPNET-Core) shows how to have strongly-typed configuration via `IOptions<T>`
- And the [relevant MS documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options) that goes into additional detail
- [This post](https://www.strathweb.com/2016/09/strongly-typed-configuration-in-asp-net-core-without-ioptionst/) covers using POCO configuration without any dependency on `Microsoft.Extensions.Options`

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

Configuration data is hierarchical: `"zxy"` is a section and `"http"` and `"nng"` are subsections of it.

Need to copy `appsettings.json` to the output folder.  To the project file add:
```xml
<ItemGroup>
    <None Update="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

In `Program.cs` add configuration:
```csharp
var host = new HostBuilder()
.ConfigureHostConfiguration(configBuilder => {
    configBuilder.AddJsonFile("appsettings.json");
    configBuilder.AddJsonFile("local.json", optional: true);
    configBuilder.AddCommandLine(args);
})
```

Here we add three "providers": two JSON files and command line arguments.  Configuration sources are processed in the order specified, in this case starting with `appsettings.json`.  That is overridden by values from `local.json`, [if it exists](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.jsonconfigurationextensions.addjsonfile#Microsoft_Extensions_Configuration_JsonConfigurationExtensions_AddJsonFile_Microsoft_Extensions_Configuration_IConfigurationBuilder_System_String_System_Boolean_) (it could be local preferences that shouldn't be checked into SCC).  Finally, command line arguments take highest precedence.

`AddCommandLine()` allows us to override settings from the command line.  For example, if using Visual Studio Code in `launch.json`:
```json
"configurations": [
    {
        "args": ["--zxy:http:port=9001"],
```

In particular, note that the "section" and "key" values are separated by "__`:`__".

Since we want to treat our services as self-contained plugins, we don't provide specific `IOption<>` configuration types via DI:
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
            // Set the listening port using the `zxy:http:port` value
            var port = config.Port;
            options.Listen(IPAddress.Loopback, port);
        })
```

Each service has a dependency on the configuration root.  It can now have its own configuration section (i.e. `zxy:http`) that is accessible in a type-safe way (via `Config` instance).

Alternatively, individual values can be loaded with the longer `configuration.GetSection("zxy:http").GetValue<int>("port")`.

Note that `WebHost` can also be implicitly configured because the default WebHostBuilder [looks for `urls` and other values](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host#server-urls) in the configuration.

## Logging

The logging system is likewise extensive.  Definitely read ["Logging in ASP.NET Core"](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/).

Bring extension methods into scope:
```csharp
using Microsoft.Extensions.Logging;
```

Configure logging and add "providers" that display or otherwise process logging:
```csharp
    var host = new HostBuilder()
    .ConfigureLogging((hostingContext, loggingBuilder) => {
        loggingBuilder.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
        // Add providers
        loggingBuilder.AddConsole();
        loggingBuilder.AddDebug(); // Visual Studio "Output" window
        loggingBuilder.AddEventSourceLogger();
    })
```

The "event source" option is interesting because it allows you to use [PerfView](https://github.com/Microsoft/perfview).  Also need to add package:
```
dotnet <project_name> add package Microsoft.Extensions.Logging.EventSource
```

Can add [configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/index#configuration) to `appsettings.json` to configure the default and per-category (e.g. `"System"`) logging levels:
```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Debug",
            "System": "Information",
        }
    }
}
```

You can get a logger instance `ILogger<T>` via DI (`T` is the log category), and log messages with the various `Log{LogLevel}()` methods:
```csharp
public class NngBroker : BackgroundService
{
    public NngBroker(ILogger<NngBroker> logger)
    {
        this.logger = logger;
        logger.LogInformation("Broker started");
    }

    protected override async Task ExecuteAsync(CancellationToken token)
    {
        //...
    }

    readonly ILogger<NngBroker> logger;
}
```

[Log messages use a template](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/index#log-message-template) for "semantic logging".  Note that it uses optionally named positional placeholders:
```csharp
// OK
logger.LogInformation("{msg} {topic:x}", msg, topic);
logger.LogInformation("{Message} {Topic:x}", msg, topic);
logger.LogInformation("{} {:x}", msg, topic);
// Works, but confusing; output is actually $"{msg} {topic}"
logger.LogInformation("{topic} {msg:x}", msg, topic);
// BAD
logger.LogInformation("{0} {1:x}", msg, topic);
```

But wait, there's more:
- The logging functions have [overloads that accept an event ID](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/indexlog-event-id)
- [Log scopes](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/index#log-scopes)
- Xunit provider (to get output when running tests): [this SO](https://stackoverflow.com/questions/46169169/net-core-2-0-configurelogging-xunit-test), [this blog](https://www.neovolve.com/2018/06/01/ilogger-for-xunit/)

## Nng

We've been using [ZeroMQ](http://zeromq.org/) for communication.  Actually, [NetMQ](https://github.com/zeromq/netmq/) along with [NetMQ.WebSockets](https://github.com/NetMQ/NetMQ.WebSockets) for [WebSocket](https://en.wikipedia.org/wiki/WebSocket) support (for our HTML5-based UI).

Our overlay and input-hooking systems (C++) moved to [nanomsg](https://github.com/nanomsg/nanomsg) for IPC/named-pipe support.  We also noted it has ["native" support for WebSocket](https://nanomsg.org/v1.1.4/nn_ws.html) and would be a good replacement for ZeroMQ.  We did some initial investigation accessing nanomsg from C#; [we forked](https://github.com/zplus/NNanomsg) [NNanomsg](https://github.com/mhowlett/NNanomsg) to convert it to .Net Standard.

Apparently [NNG (nanomsg-next-generation)](https://github.com/nanomsg/nng) is being developed as the successor to nanomsg (with the latter now in "maintinence mode").  For C#, [we forked](https://github.com/zplus/csnng) [csnng](https://github.com/zplus/csnng) (again, to convert it to .Net Standard and fix some issues), but have since abandoned it for our own [nng.NETCore](https://github.com/subor/nng.NETCore).

Here's a background service that creates a simple NNG broker.  It also nicely illustrates everything together:
```csharp
class NngConfig
{
    public string BrokerIn {get; set;}
    public string BrokerOut {get; set;}
}

public class NngBroker : BackgroundService
{
    public NngBroker(ILogger<NngBroker> logger, FactoryType factory, IConfiguration configuration)
        {
            this.logger = logger;
            this.factory = factory;

            config = new NngConfig();
            var nngSection = configuration.GetSection("zxy:nng");
            nngSection.Bind(config);

            logger.LogInformation("Broker started");
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
            logger.LogInformation("Broker: {} {:x}", msg, topic);
            if (!await output.Send(msg))
            {
                await Console.Error.WriteLineAsync("Failed!");
            }
        }
    }

    readonly ILogger<NngBroker> logger;
    readonly FactoryType factory;
    readonly NngConfig config;
}
```

Where `FactoryType` is dependency for creating nng resources [via NNG native assemly]({% post_url /2018/2018-09-09-native-assembly %}#one-load-context to-rule-them-all) (see [nng.NETCore source](https://github.com/subor/nng.NETCore/blob/master/nng.Shared/AssemblyLoadContext.cs)):
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

We've got a good start for a new [layer0]({% post_url /2018/2018-08-28-layer0 %}): microservices loaded from plugins, HTTP server for HTML5 UI, configuration and logging, communication layer, etc.  Still need to investigate how we can simplify [layer1]({% post_url /2018/2018-09-04-layer1 %}) management, whether we use Thrift or something else for serialization, and a few other key facets, but there's already a lot to like.

Microsoft was hardly the first DI solution and most likely isn't the best, but it plays nicely with ASP.[]()NET Core, .NET Core in general, and the rest of the Microsoft ecosystem.  In any case, I think it serves as a great replacement for our in-house microservices application scaffolding.