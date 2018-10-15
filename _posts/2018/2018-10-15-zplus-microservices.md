---
layout: post
title: The Z+ System Software
tags:
- microservices
- architecture
- zplus
---

At ChinaJoy 2018 (end of July) we announced "__Z+__", a dual-purpose PC and game console based on a semi-custom AMD [SoC](https://en.wikipedia.org/wiki/System_on_a_chip) (Ryzen CPU and Vega GPU similar to [Raven Ridge](https://en.wikipedia.org/wiki/Ryzen#Raven_Ridge)).  Additional info:
- [Anandtech](https://www.anandtech.com/show/13163/more-details-about-the-zhongshan-subor-z-console-with-custom-amd-ryzen-soc)
- [Eurogamer](https://www.eurogamer.net/articles/digitalfoundry-2018-hands-on-with-subor-z-plus-ryzen-vega-chinese-console)
- [Our website](http://www.playzplus.com/) (Simplified Chinese)
- [Famitsu](https://www.famitsu.com/news/201808/05161964.html) (Japanese)

Within the next month we'll ship it as a PC, and Q1 2019 we'll release the game console functionality.

At the start of the project we had several high-level goals in mind for the system software:  
- UI completely decoupled from business logic, multiple communicating clients
- Multi-core friendly, asynchronous [all the way down](https://en.wikipedia.org/wiki/Turtles_all_the_way_down)
- Portable; multi-platform friendly
- Modular and loosely coupled to simplify testing and enable [feature flags](https://en.wikipedia.org/wiki/Feature_toggle)

While internally I intentionally avoided using the "microservices" buzzword because it tends to focus on server-side architecture, that's very much how our system software is designed.  We spent a few months working on a prototype and almost 1.5 years in full production ramping up to 10 full-time software engineers working on the system software.  What follows is equal parts architectual overview and post-mortem for what is still "beta" software.

## Cast

[Our Github documentation](https://github.com/subor/sdk/blob/master/docs/README.md) has a rough [platform diagram](https://github.com/subor/sdk/blob/master/docs/topics/layer0.md) but we need to update it with something more detailed and accurate:  
![]({{ "/assets/zplus_arch.png" | absolute_url }})

The machine runs Windows 10 IoT Enterprise (no relation to "[IoT Core](https://docs.microsoft.com/en-us/windows/iot-core/windows-iot-core)", confusingly).

[Layer0]({% post_url /2018/2018-08-28-layer0 %}) is a [Windows service]({% post_url /2018/2018-08-21-windows-services %}) functioning as the microservice container.  [Layer1]({% post_url /2018/2018-09-04-layer1 %}) is a child container with additional services that need access to the user desktop.

The services themselves are [MEF extensions]({% post_url /2018/2018-08-16-mef1-mef2 %}).  Communication is done via Thrift RPC as well as [ZeroMQ]({% post_url /2018/2018-09-07-zeromq-thrift %}) for [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern).

Some examples of services:  

| Name | Layer | Description
|-|-|-
| Online | 0 | Connection to Braincloud and bridge between our platform and their REST API
| User | 0 | User session management: binding users to input devices, managing game/user/platform connections to online services
| Settings | 0 | User and system settings/configuration
| Input | 1 | Manages input devices (especially gamepads)
| Overlay | 1 | Communication with and management of overlay, displaying pop-up notifications, etc.
| Power & Light | 1 | Handle power button events, transitions to/from LPM, LED control

There's a few different clients, but the most important ones are:
-  Main client uses [Chromium Embedded Framework (CEF)](https://bitbucket.org/chromiumembedded/cef) to render our HTML5-based UI ([Vue.js](https://vuejs.org/))
- Overlay is based off tech licensed from [Evolve](https://www.evolvehq.com) and handles displaying UI (also CEF/HTML5) over apps/games

The backend is a customized instance of [Braincloud](https://getbraincloud.com/) running atop [AliYun/"AliCloud" PaaS](https://www.alibabacloud.com/) (very similar to AWS).

## The Good

### Granularity

Initiallly there was some worry about being too fine-grained and excessive communication overhead ("chatty" services).  A team member asked, "how small should we make the services?"  To which I replied "if two modules only ever talk to each-other they should be in the same service".  Was a flippant remark at the time, but turned out to work well in practice.

### Client Modularity

We were able to build several clients: main client, overlay, minipower ([WPF](https://docs.microsoft.com/en-us/dotnet/framework/wpf/getting-started/introduction-to-wpf-in-vs)), [example](https://github.com/subor/sample_unity_space_shooter) [games](https://github.com/subor/sample_ue4_platformer), etc.  We could pick the right tool for the job and support a broad range of 3rd-party applications.

Our [Low Power Mode (LPM)](https://github.com/subor/sdk/blob/master/docs/topics/lpm.md) power-gates the SoC so only a single CPU core is on and the memory/GPU is clock-gated.  This is intended for doing background downloads/updates and the system consumes around 35W (we originally targetted ~20W, but that didn't pan out).  For comparison: under normal operation the system idles at 60W, running a game like _Rise of the Tomb Raider_ is upwards of 180W, and a CPU/GPU stress-test can hit 220W.  Layer1, main client, and overlay can all be shutdown leaving a fairly minimal layer0 for basic system functionality.

### Service Modularity/Isolation

When we converted layer0 to a Windows service we needed to create layer1 into which we moved several modules.  We moved 4 services out of layer0 and into layer1- a completely seperate process.  It took around a week and that was mostly dealing with spawning and managing a second process.
In general, services can (and do) fail and it doesn't take the entire application down.

### Parallelism

Owing to each service getting it's own thread(s) and judicious use of the thread pool we make good use of all 4 CPU cores and 8 [hardware threads](https://en.wikipedia.org/wiki/Simultaneous_multithreading).

Background uploads/downloads, latency senstive input processing, UI updates, and interacting with online services all happen simultaneously without issue.

### Code Reuse/Platform Agnostic

Layer0 runs on Windows 7 to Windows 10 and we've got prototypes running on IoT Core, OSX via Mono, and Android (and probably iOS) via [Xamarin](https://docs.microsoft.com/en-us/xamarin/).

## The Bad and/or Ugly

### Convoluted Startup/Shutdown Process and Service Lifetime

This wasn't immediately clear at the onset and then evolved somewhat organically over time.

### Reinvented Dependency Injection- Poorly

`static` classes?  Check.  Singletons?  Check.  Passing contexts/factories around?  Check.  "Mini plugins" for platform-specific code within services?  Check.  Some actual dependency injection?  Naturally.

### Overhead and Resource Usage

Last census we have 24 services and 48 binaries (executables and libraries- excluding tests and 3rd-party/framework dependencies).  Startup time is a bit worse than desired.

First implementation used threads- 2 per service.  24 services, and well... you do the math.  Including the non-service components we are spawning something like 60 threads.  Granted, most of which are blocked in `accept()` and the like.  We did some of the leg-work moving everything to `Task` and the Task Parallel Library, but this isn't finished.

Moving to two processes necessitated switching ZeroMQ from `inproc://` to `tcp://` because it doesn't support named pipes.  I think we're using around 100 ports...  Keep in mind this is after using `TMultiplexedProtocol`.

At first glance this probably doesn't seem that bad.  But, keep in mind our platform isn't the main application, we need to preserve that majority of hardware resources for games.

### Redundant Layers and Indirection

For historical reasons, messages from layer0/layer1 first go through C#/CEF portion of main client and are then forwarded to hosted HTML5/Javascript.

Each service has an internal and external interface.  This was intended to provide for internal and external APIs, what was used by us vs. external developers, respectively.  Unfortunately, it manifested itself in the form of an additional abstraction layer (and additional threads and connections).

### Self-inflicted Dependencies and Rigidity

Several of the services wait for another service to start.  Consequently, if that service fails basically nothing starts.  We've got a few other microservice anti-patterns like (relatively short) chains of synchronous request/response calls between services.  While we could have used ZeroMQ's pub/sub for asynchronous requests and [integration events](https://github.com/dotnet/docs/blob/master/docs/standard/microservices-architecture/multi-container-microservice-net-applications/integration-event-based-microservice-communications.md), in practice we didn't.

## Conclusion

In short, we achieved some of our goals, but failed to capitalize on all the benefits of a microservice architecture- particularly the autonomy.  This was mostly a result of our unfamiliarity and inexperience with the pattern.  However, overall, I believe the majority of the team would regard this experiment as a success.

This was pretty high level and I glossed over most of the details, but I intend to use it as a starting point for talking about the "second generation" of our system software.