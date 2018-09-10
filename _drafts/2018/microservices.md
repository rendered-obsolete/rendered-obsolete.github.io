---
layout: post
title: 
tags:
- walloftext
- architecture
---

Every project I've ever been on the UI becomes a nightmare.

Multi-core friendly.

Spent a few months working on a prototype and 1.5 years in full production ramping up to 10 full-time software engineers.

## Actors

ZeroMQ
Low Power Mode (LPM)
Main Client

| Name | Layer | Description
|-|-|-
| Input | 1 | Managees input devices (keyboards and gamepads)
| Power & Light | 0 | Transitions to/from LPM, LED control
| Online | 0 | Connection to Braincloud
| User | 0 | User session management: binding users to input devices, managing game/user/platform connections to online services

## The Good

- Granularity

At the onset we worried about being too fine-grained and excessive communication overhead.  A team member asked, "how small should we make the services?"  To which I replied "if two modules only ever talk to each-other they should be in the same service".

- Client modularity

We were able to build several clients: main client, overlay, minipower, example games, etc.

- Service modularity/independence

When we converted layer0 to a Windows service we needed to create layer1 into which we moved several modules.  We moved 4 services out of layer0 and into layer1- a completely seperate process.  It took around a week and that was mostly dealing with spawning and managing a second process.
Services can and do fail and it doesn't take the entire application down.

- Parallelism

Owing to each service getting 

## The Bad and/or Ugly

- Startup/shutdown is convoluted

This evolved somewhat organically over time.

- Reinvented depedency injection.  Poorly.

`static` members?  Check.  Singletons?  Check.  Passing contexts/factories around?  Check.  Mini plugins for platform-specific code within services?  Check.  Some actual dependency injection?  Naturally.

- Overhead

Last census the background daemon has 24 services and 48 binaries (executables and libraries- excluding tests and 3rd-party dependencies).

- Redundant layers/indirection

For historical reasons, messages from layer0/layer1 first go through C#/CEF layer of main client and are then forwarded to hosted HTML5/Javascript.

Each service has an internal and external interface.  This was intended to provide for internal and external APIs, what was used by us vs. external developers, respectively.  Unfortunately, it manifested itself in the form of discrete threads and connections.

- Resource usage

First implementation used threads.  And we were creating 2 per service.  24 services, and well... you do the math.  Including the non-service components and we were spawning something like 60 threads- most of which are blocked in `accept()`, etc.  I did some of the leg-work moving everything to `Task`.
Moving to two processes necessitated switching ZeroMQ from `inproc://` to `tcp://` because it doesn't support named pipes.  I think we're using around 100 ports...  Keep in mind this is after using `TMultiplexedProtocol`.

- Self-inflicted dependencies and rigidity

Several of the services wait for another service to start.  Consequently, if that service fails basically nothing starts.

Oh, and we've got an `enum` listing all the services.