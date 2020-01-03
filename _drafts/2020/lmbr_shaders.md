---
layout: post
title: Lumberyard Shader Pipeline
tags:
- lumberyard
- gamedev
- pipeline
- shader
- graphics
canonical_url: 
series: Amazon Lumberyard
---

If asked to pick the defining characteristic of a game engine, the rendering pipeline could win a popularity contest- the star of which are the programmable shaders used by modern GPUs.  Here we'll look at part of the data pipeline shaders go through to reach the Lumberyard runtime.  In particular the "shader cache" and __Remote Shader Compiler__ which are often a pain point for new developers.

## Shaders

All the shaders available in the __Material Editor__ are defined in `dev/Editor/Materials/ShaderList.xml`.  Each of which has a corresponding extension file `dev\Engine\Shaders\*.ext` containing shader-specific material permutation flags.  Additionally, `Runtime.ext` contains runtime permutation flags shared by all shaders.  These permutation flags are applied to "CryFX" shader source code in `dev\Engine\Shaders\HWScripts\CryFX\` to generate uber shader permutations.

[Custom shaders](https://docs.aws.amazon.com/lumberyard/latest/userguide/mat-shaders-custom-dev-intro.html) are out of scope, but exemplifies this.

## Shader Cache

["Shader cache" refers to a variety of files](https://docs.aws.amazon.com/lumberyard/latest/userguide/mat-shaders-custom-dev-cache-intro.html) that are produced by processing the CryFX source.

Depending on the platform and type of build you may have:
- `Shaders.pak` - shader source code (from `dev\Engine\Shaders\HWScripts\CryFX\`)
- `ShadersBin.pak` - binary parsed shader source
- `ShaderCache.pak` - compiled shaders
- `ShaderCacheStartup.pak` - subset of shader cache used during startup (i.e. to show a loading screen)

We'll talk about the last two when we look at authoring a release binary, but for now it's worth mentioning they're created by `lmbr_pak_shaders.bat`.

If you open `shadercache.pak` you will find a series of folders for each shader type:
```
└───shaders
    └───cache
        └───D3D11
            │   ...
            │   postaa.cfxb
            │   postaa.fxb
            │   posteffects.cfxb
            │   posteffects.fxb
            │   posteffectsgame.cfxb
            │   posteffectslib.cfib
            │   ...
            │
            ├───cgcshaders
            │
            ├───cgdshaders
            │
            ├───cghshaders
            │
            ├───cgpshaders
            │       ...
            │       postaa@blendweightsmaa_ps.fxcb
            │       postaa@debugpostaa_ps.fxcb
            │       postaa@fxaa_ps.fxcb
            │       postaa@lumaedgedetectionsmaa_ps.fxcb
            │       postaa@neighborhoodblendingsmaa_ps.fxcb
            │       postaa@postaacomposites_ps.fxcb
            │       postaa@postaamotiondebug_ps.fxcb
            │       postaa@smaa_1tx_ps.fxcb
            │       postaa@taa_ps.fxcb
            │       postaa@upscaleimageps.fxcb
            │       ...
            │
            └───cgvshaders
                    ...
                    postaa@basevs.fxcb
                    postaa@postaa_vs.fxcb
                    ...
```

Once you run the game, in `user\cache\` you will likewise find a very similar `shaders\cache\d3d11\`:
```
│   lookupdata.bin
│
├───cgcshaders
│
├───cgdshaders
│
├───cggshaders
│
├───cghshaders
│
├───cgpshaders
│       ...
│       postaa@taa_ps.fxcb
│       ...
│
└───cgvshaders
```

| Extension | Description | Details
|-|-|-|
| `cfx`, `cfi` | CryFX shader source/include | See `dev\Engine\Shaders\HWScripts\CryFX\`
| `cfxb`, `cfib` | CryFX (include) binary | `Code\CryEngine\RenderDll\Common\Shaders\CShaderBin.h`, see `SShaderBinHeader`
| `fxb` | Pre-parsed shader | Resource file (`Code\CryEngine\RenderDll\Common\ResFile.h`) containing a shader (`Code\CryEngine\RenderDll\Common\Shaders\ShaderSerialize.h`)
| `fxcb` | Another shader binary | ?


| Abbreviation | Shader Type |
|-|-|
| `VS` | vertex shader
| `HS` | hull shader (tessellation)
| `DS` | domain shader (tessellation)
| `GS` | geometry shader
| `PS` | pixel/fragment shader

The `ShaderList_*.txt` file it's generating contains output like:
```
<4>FixedPipelineEmu@FPPS()()(2a2a0505)(0)(0)(0)()(PS)
<4>PostAA@TAA_PS()(%_RT_SAMPLE1|%_RT_SAMPLE2|%_RT_SAMPLE3)(0)(0)(0)(0)()(PS)
```

These "request lines" describe a shader permutation that was requested by the runtime.  "PostAA" in the second line refers to `dev\Engine\Shaders\HWScripts\CryFX\PostAA.cfx`.  There is a function `TAA_PS()` and there are conditional compilation blocks like `#if %_RT_SAMPLE1`.  The shader type comes at the end, `PS` means it's a pixel (fragment) shader.


Over the course of development you'll experiment with different effects, add/remove assets, and otherwise pull in other shader permutations that will pollute the shader cache.  Once you're closer to shipping you'll want to purge the shader list so it's closer to the minimum number of shaders you actually need as opposed to the super-set of all shaders that were ever used.  Another approach is separating the remote shader compiler used by your developers and QA, the latter will be composed of only those shaders needed for the release build and no include those only needed for development, debugging, etc.

You might be thinking:

> This doesn't seem very deterministic...

Yeah, it's not.


Lumberyard's shaders are found in `dev\Engine\Shaders\HWScripts\CryFX\`.  The "CryFX" `*.cfx` files are mostly written in [HLSL](https://en.wikipedia.org/wiki/High-Level_Shading_Language) with some additional pre-processing/macros.

If RSC complains:
```
Warning: unauthorized IP 10.151.67.44 trying to connect. If this IP is authorized please add it to the whitelist in the config.ini file
```

Add `dev\Tools\CrySCompileServer\x64\profile`:
```
white_list=10.151.0.0/16
```