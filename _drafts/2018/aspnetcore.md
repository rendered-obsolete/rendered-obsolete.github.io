---
layout: post
title: Migrating to ASP.NET Core w/ Apache Thrift
tags:
- thrift
- aspnetcore
- csharp
- netcore
---

The bulk of our client UI is HTML5 (via [CEF](https://bitbucket.org/chromiumembedded/cef)) which uses [Apache Thrift](https://thrift.apache.org/) to talk to a Windows service (via HTTP and WebSockets).  As part of our migration to .NET Core we set out to:
- Use the new `netcore` generator in [thrift 0.11](https://github.com/apache/thrift/releases/tag/0.11.0)
- Handle HTTP requests with `Thrift.Transports.Server.THttpServerTransport` atop [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/) instead of `System.Net.HttpListener` handing requests to `Thrift.Transport.THttpHandler`

## Before- HttpListener

The original implementation based on `System.Net.HttpListener` was similar to:
```csharp
if (!System.Net.HttpListener.IsSupported)
{
    return;
}

// MultiProcessors.HttpProcessor is a Thrift.TMultiplexedProcessor
var thrift = new Thrift.Transport.THttpHandler(MultiProcessors.HttpProcessor, new Thrift.Protocol.TJSONProtocol.Factory());
var listener = new System.Net.HttpListener();
listener.Prefixes.Add("http://localhost:8282/");
listener.Start();

while (!cts.IsCancellationRequested)
{
    // Receive HTTP request
    var ctx = await listener.GetContextAsync();
    await Task.Run(() =>
    {
        try
        {
            // FIXME: implement CORS correctly
            ctx.Response.AppendHeader("Access-Control-Allow-Origin", "*");

            if (ctx.Request.HttpMethod == "OPTIONS")
            {
                ctx.Response.AddHeader("Access-Control-Allow-Headers", "Content-Type, Accept, X-Requested-With");
                ctx.Response.AddHeader("Access-Control-Allow-Methods", "Get, POST");
                ctx.Response.AddHeader("Access-Control-Max-Age", "1728000");
                ctx.Response.Close();
            }
            else
            {
                // Pass request to thrift services registered with multiplexed processor
                thrift.ProcessRequest(ctx);
            }
        }
        catch(Exception e)
        {
            Log(LogLevel.Warn, $"HttpListener Exception: {e.Message}");
        }
    });
}
```

The CORS hack needed for CEF to load content directly from disk.

## After- ASP.[]()NET Core

As I started looking into ASP.[]()NET Core the level of configurability and sheer sophistication was pretty daunting.  More importantly, it wasn't immediately clear how to handle HTTP requests with thrift.

The MS documentation is extensive.  The following will help you get started:
- [ASP.NET Core Web Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host)
- [Application Startup in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup)

The following was cobbled together from numerous sources:
```csharp
var factory = new Microsoft.Extensions.Logging.LoggerFactory();

try
{
    webHostBuilder = WebHost.CreateDefaultBuilder()
    .UseKestrel(options =>
    {
        options.Listen(IPAddress.Loopback, 8282);
    })
    //.UseUrls("http://localhost:8282/")
    //.UseStartup<Startup>()
    .ConfigureServices(services =>
    {
        services.AddCors();
    })
    .Configure(app =>
    {
        app.UseCors(corsPolicyBuilder =>
        {
            corsPolicyBuilder.AllowAnyHeader();
            corsPolicyBuilder.AllowAnyOrigin();
            corsPolicyBuilder.AllowAnyMethod();
        });
        // Here's our thrift middleware
        app.UseMiddleware<Thrift.Transports.Server.THttpServerTransport>(MultiProcessors.HttpProcessor, new TJsonProtocol.Factory());
    })
    .Build();
    webHostBuilder.Start();
}
catch (Exception ex)
{
    log(LogLevel.Error, "HTTP server failed: " + ex.Message);
}
```

This uses the [ConfigureServices() and Configure() helpers instead of a Startup class](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup#convenience-methods).

Rather than waiting for a routine to return an HTTP request and then passing it to a handler, ASP.[]()NET Core can be configured with ["middleware"](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/index) to handle requests and process responses.

I initially was trying to add thrift as a service via [ASP.NET Core dependency injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection):
```csharp
.ConfigureServices(services =>
{
    // This doesn't work
    services.AddSingleton<ITAsyncProcessor>(MultiProcessors.HttpProcessor);
    services.AddSingleton<ITProtocolFactory>(new TJsonProtocol.Factory());
})
```

But got errors like:
```
Unable to resolve service for type 'Thrift.ITAsyncProcessor' while attempting to activate 'Thrift.Transports.Server.THttpServerTransport'.
```

Services probably aren't the correct mechanism, while middleware seems intended for request/response handling.

The difference between `AddSingleton()`, `AddTransient()`, etc. pertains to the [lifetime of the service](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection#service-lifetimes).

Finally, make sure to call `webHostBuilder.StopAsync()` when finished.  Otherwise you'll get a native exception in the GC finalizer.

## Conclusions

Unfortunately, we don't have any perfomance tests for this area of code.  Because I would have loved to see the results after learning [what a beast ASP.NET Core is](https://www.ageofascent.com/2016/02/18/asp-net-core-exeeds-1-15-million-requests-12-6-gbps/).

We've got a first version working, but I don't fully understand it yet.  The [same architecture can be used to bulid Net Core console applications](https://odetocode.com/blogs/scott/archive/2018/08/16/three-tips-for-console-applications-in-net-core.aspx), so it would be well-worth investing more time.