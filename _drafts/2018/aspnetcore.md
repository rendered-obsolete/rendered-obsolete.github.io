---
layout: post
title: Migrating from MEF1 to MEF2
tags:
- thrift
- aspnetcore
- csharp
---

The bulk of our client UI is HTML5 via [CEF](https://bitbucket.org/chromiumembedded/cef) which talks to a Windows service using [Apache Thrift](https://thrift.apache.org/) through HTTP and WebSockets.  As part of our migration to .NET Core:
- Use the new `netcore` generator in [thrift 0.11](https://github.com/apache/thrift/releases/tag/0.11.0)
- Handle HTTP requests with `Thrift.Transports.Server.THttpServerTransport` atop ASP.NET Core instead of `Thrift.Transport.THttpHandler` and `System.Net.HttpListener`

The original implementation based on `System.Net.HttpListener` looked similar to:
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

The CORS hack was needed for CEF to load content directly from disk.

As I started looking into ASP.NET Core the level of configurability and sheer sophistication was pretty daunting.  More importantly, it was immediately clear how to pass HTTP requests to thrift.


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
        // Middleware:
        //https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-2.1&tabs=aspnetcore2x
        // Read https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.1 when get error like:
        //Unable to resolve service for type 'Thrift.ITAsyncProcessor' while attempting to activate 'Thrift.Transports.Server.THttpServerTransport'.
        //services.AddSingleton<ITAsyncProcessor>(MultiProcessors.HttpProcessor);
        //services.AddSingleton<ITProtocolFactory>(new TJsonProtocol.Factory());
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

[what a beast ASP.NET Core is](https://www.ageofascent.com/2016/02/18/asp-net-core-exeeds-1-15-million-requests-12-6-gbps/)