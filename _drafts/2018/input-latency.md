---
layout: post
title: Gamepad Input with Rust
tags:
- rust
- input
- talesfromthecrypt
- story-time
---

## Side-Quest

In a previous life I worked in real-time computing laboratories.

At the time (perhaps still) there was a distinction between "hard" and "soft" realtime systems; where people died if time contraints weren't met (think planes, trains, and automobiles), and where they didn't, respectively.  My professor, Dr. Niehaus, liked to talk about "firm" realtime systems- where the value diminished and rapidly approached zero as results were late.  Namely, multi-media.

In my second life, we were working with [UE3](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_3) and ["Measuring Responsiveness in Video Games"](http://www.gamasutra.com/view/feature/3725/measuring_responsiveness_in_video_.php?print=1) became a talking point.  Our input latency wasn't great.  Incidentally, I happen to also have worked at the company that did the "PS3 System menus" and "PixelJunk Racers" in the 60 fps category.  I knew how "thin" their engine was.  However, in UE3- like many other commercial game engines from that era- input is handled and drives gameplay on a "main" thread, which pushes work to a "rendering" thread.  

On the [PS3](https://en.wikipedia.org/wiki/PlayStation_3), this then kicked off jobs to the [Cell's SPUs](https://en.wikipedia.org/wiki/Cell_%28microprocessor%29) (which did computationally expensive work like [back-face culling](https://en.wikipedia.org/wiki/Back-face_culling) and triangle reordering, generating [pixel shaders](https://en.wikipedia.org/wiki/Shader#Pixel_shaders), [skinning](https://en.wikipedia.org/wiki/Skeletal_animation) and [morph targets](https://en.wikipedia.org/wiki/Morph_target_animation), and certain rendering post-processing).  They then write GPU commands into a queue, which are read by the GPU into an buffer, which go through the GPU pipeline, ending up pixels written to a backbuffer.  Which is eventually flipped to the on-screen image.

*phew*

End result is fantastic parallelism; full hardware utilization in multi/many-core and [heterogenous architectures](https://en.wikipedia.org/wiki/Heterogeneous_System_Architecture) (most common being CPU with GPU).  At the cost of latency; as we add stages the distance between button press to on-screen pixels increases.

When I think of the double-edged sword that is deep pipelines I think of the Pentium 4.  There were other things at play like [hazards](https://en.wikipedia.org/wiki/Hazard_(computer_architecture)) and [branch prediction](https://en.wikipedia.org/wiki/Branch_predictor), but it's a case-study in the risk-factor involved.

__"Don't Cross the Streams" and also "Never Wait"__

One of the bugs I'm least fond of is easy to reproduce in the [PS3 version of "Alice: Madness Returns"](https://www.metacritic.com/game/playstation-3/alice-madness-returns).  Run around a few minutes and switch weapons.  Framerate drops.  Every time.

Without getting into the details why, we stored largish audio and animation data ([LZMA compressed](https://en.wikipedia.org/wiki/Lempel%E2%80%93Ziv%E2%80%93Markov_chain_algorithm)) in VRAM and used the GPU to push it into system memory when needed.  This was high latency (round-trip was on the order of 100 ms- a few frames at 30 fps) but also high bandwith.  There were a number of mitigations in place to deal with the latency:
- 8 MB [LRU cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) in system memory
    - This was empirically determined to be the largest buffer we could afford without succombing to memory fragmentation
- Data "fetch" commands were batched and given higher priority than rendering commands
    - Game thread inserted them on the front of the render thread command queue
    - GPU commands were slipped into holes in GPU command buffer that were set aside for output [DMA](https://en.wikipedia.org/wiki/Direct_memory_access)'d from SPU jobs
- Aggressive pre-fetching:
    - All the game levels have audio triggers that play different, random background sound effects.  Because they were small, we pre-load all the sfx needed when we loaded a trigger.
    - Same for skeletal mesh anim sets

Which _mostly_ works.  Except when there's no warning.  Namely things we can't predict- like user input that switches weapons (requiring new animation and sound sets).  Trying to use data that isn't completely in memory (yet) would be bad.  So, we do the only thing that could be shoe-horned in- the CPU waits on the GPU.

__Neurosis__

In my current life I would describe myself as "latency sensitive".  We primarily use C#, but to get the latency as low as possible (and, more importantly, avoid any hitches caused by [GC](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)).