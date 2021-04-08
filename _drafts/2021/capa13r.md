---
layout: post
title: Ryzen SBC
tags:
- sbc
- amd
---

Between my love of game consoles and working on [Z+ with AMD]({% post_url /2018/2018-10-15-zplus-microservices %}) I perhaps have an unhealthy obsession with AMD APUs.  When AxiomTek announced a SBC with Vega graphics and Zen CPU cores I was... intrigued:

- [News 1](https://www.techradar.com/news/raspberry-pi-not-powerful-enough-for-you-this-compact-board-boasts-an-amd-ryzen-apu)
- [News 2](https://techbriefly.com/2020/05/20/the-more-powerful-raspberry-pi-alternative-carries-an-amd-ryzen-apu/)

I picked up a CAPA13R (V1605B @2.0 GHz) I'm far from an expert in these things, but I had in mind to turn it into a sort of Steam box for my TV and added the following:

- Transcend 240GB SATA III 6Gb / s MTS420S 42mm M.2 SSD
- HyperX Impact DDR4 16GB, 2666MHz CL15 SODIMM XMP - HX426S15IB2 / 16
- PicoPSU -90 12V DC-DC ATX Mini-ITX 0-90W power supply power
- Salcar 60 W Transformer Power Adapter (12 V 6 A)

Until [ATX12VO](https://en.wikipedia.org/wiki/ATX#ATX12VO) (also see [this guide](https://www.gamersnexus.net/guides/3568-intel-atx-12vo-spec-explained-what-manufacturers-think))ol becomes a thing, I'm using a 90W PicoPSU with pins 4 and 5 shorted to boot the system.  Depending on your power supply of choice, another approach may be necessary- Google is your friend.

With a stock install of Windows 10 20H2/19042.631 here's benchmark numbers:

Before you do anything install the [drivers](https://www.axiomtek.com/Default.aspx?MenuId=Products&FunctionId=ProductView&ItemId=25592&C=CAPA13R&upcat=270) to get audio and graphics perfomance working.

In general I stuck with the defaults:

Benchmark | Settings | Score Min/Avg/Max FPS
-|-|-
Unigine: Heaven 4.0 | 4K/OpenGL | 45 1.1 1.8 4.0
Unigine: Heaven 4.0 | 1920x1200/OpenGL | 252 4.5 10.0 24.1
Unigine: Heaven 4.0 | 1080p/OpenGL | 317 5.1 12.6 35.1
Unigine: Superposition v1.1 | 1080p med/OpenGL | 857 5.72 6.41 7.89
Unigine: Valley 1.0 | 1920x1200/DX11 | 596 9.7 14.2 24.1

Benhmmfk | Settings | Score Graphics CPU
3DMark Demo Time Spy v1.2 | 4K/64-bit | 595 529 2085
3DMark Demo Time Spy v1.2 | 1920x1200/64-bit | 641 571 2170

So, seemingly not cut out for 4K gaming, however, during none of the benchmarks did the heatsink even become warm.  And even the 1080p performance it's very good.  I suspect either the PicoPSU or power adapter just wasn't delivering enough power.

AMD V1605B @2.0 GHz 16GB DDR4, min. RMS. +12V@2.5A
AMD V1807B @3.35 GHz 16GB DDR4, min. RMS. +12V@4.2A