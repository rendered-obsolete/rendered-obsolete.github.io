---
layout: post
title: Rewrite
tags:
- netcore
---

## Configuration

Came across this blog that showed how to use json, but didn't really seem complete:
https://dzone.com/articles/read-config-data-in-net-core-test-project-net-core

https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-2.1#override-configuration
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/index?view=aspnetcore-2.1
https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.1



In `appsettings.json`:
```json
{
    "urls": "http://*:8284",
    "http":{
        "port":8283
    }
}
```
`using Microsoft.Extensions.Configuration;`

`Main.cs`:
```csharp
;

var host = new HostBuilder()
.ConfigureHostConfiguration(configBuilder => {
    configBuilder.AddJsonFile("appsettings.json");
    //configBuilder.AddJsonFile("local.json", optional: true);
    configBuilder.AddCommandLine(args);
})
```


```csharp
public HttpXy(IConfiguration config)
{
    this.config = config;
}

protected override Task ExecuteAsync(CancellationToken token)
{
    var webHostBuilder = WebHost.CreateDefaultBuilder()
    .UseConfiguration(this.config)
```

This works because the default WebHostBuilder [looks for `urls`](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-2.1#server-urls).

Alternatively, we can use the `http:port` value I added:
```csharp
    var webHostBuilder = WebHost.CreateDefaultBuilder()
    .UseKestrel(options => {
        var port = config.GetSection("http").GetValue<int>("port");
        options.Listen(IPAddress.Loopback, port);
    })
```

`AddCommandLine()` allows us to override settings from the command line.

For example, in `launch.json`:
```json
"configurations": [
    {
        "args": ["--http:port=9001"],
```

In particular, note that the "section" and "key" values are separated by "__`:`__".

## Services

[BackgroundService](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice)