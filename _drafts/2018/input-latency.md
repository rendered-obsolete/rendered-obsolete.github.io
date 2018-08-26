---
layout: post
title: Rust & Apache Thrift
tags:
- talesfromthecrypt
- story-time
---

## Side-Quest

In a previous life I worked in real-time computing laboratories.

At the time there was a distinction between "hard" and "soft" realtime systems; where people died if time contraints weren't met (think planes, trains, and automobiles), and where they didn't, respectively.  My professor, Dr. Niehaus, liked to talk about "firm" realtime systems- where the value diminished and rapidly approached zero as results were late.  Namely, multi-media.

["Measuring Responsiveness in Video Games"](http://www.gamasutra.com/view/feature/3725/measuring_responsiveness_in_video_.php?print=1)

Our input latency wasn't great.  Like many other game engines (particularly from that era), input is handled and drives gameplay on a "main" thread, which pushes work to a "rendering" thread.

On the [PS3](https://en.wikipedia.org/wiki/PlayStation_3), this kicked off jobs to the [Cell's SPUs](https://en.wikipedia.org/wiki/Cell_%28microprocessor%29) (which did computationally expensive work like [back-face culling](https://en.wikipedia.org/wiki/Back-face_culling) and triangle reordering, generating [pixel shaders](https://en.wikipedia.org/wiki/Shader#Pixel_shaders), [skinning](https://en.wikipedia.org/wiki/Skeletal_animation) and [morph targets](https://en.wikipedia.org/wiki/Morph_target_animation), and certain rendering post-processing).  They then write GPU commands into queue, which are read by the GPU into an buffer, which go through the GPU pipeline, ending up pixels written to a backbuffer.  Which is flipped to the on-screen image.

*phew*

End result is fantastic parallelism; hardware utilization in multi/many-core and [heterogenous architectures](https://en.wikipedia.org/wiki/Heterogeneous_System_Architecture) (most common being CPU with GPU/co-processor).  At the cost of latency; as we add stages the distance between button press to on-screen pixels increases.

When I think of the double-edged sword that is deep pipelines I think of the Pentium 4.  There were other things at play like [hazards](https://en.wikipedia.org/wiki/Hazard_(computer_architecture)) and [branch prediction](https://en.wikipedia.org/wiki/Branch_predictor), but illustrates the risk-factor.

One of the bugs I'm least proud of.  Run around a few minutes and switch weapons.  Framerate drops.  Every time.

Without getting into the details why, we store largish audio and animation data ([LZMA compressed](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Markov_chain_algorithm)) in VRAM and used the GPU to push it into system memory when needed.  Number of mitigations in place:
- 8 MB [LRU cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU))
- "Data" fetch commands

Which mostly works.  Except when there's no warning.