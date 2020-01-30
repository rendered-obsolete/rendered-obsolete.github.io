---
layout: post
title: Lumberyard Asset Dependency Graph and Asset Bundler
tags:
- lumberyard
- gamedev
canonical_url: 
series: Amazon Lumberyard
---

One of the marquee features released with [Lumberyard 1.22](https://docs.aws.amazon.com/lumberyard/latest/releasenotes/lumberyard-v1.22.html) is a new [asset dependency system](https://aws.amazon.com/blogs/gametech/1-22/).  Excerpt rom the blog:

> Lumberyard’s new Asset Dependency Graph provides the means to determine the set of assets a given asset depends on. By recursively walking the entire dependency graph, the engine can easily determine the exact set of assets your game needs to run.

This has numerous potential uses:  

- Build size optimization; ignore unused assets and only package assets that are actually used
- Bundle content; additional weapons/characters/levels/etc. can be self-contained collection of all required assets
- Download/disc layout; determine the minimum content that is required to run
- Load pre-fetching; load all required assets at once rather than waiting for them to be loaded as dependencies are loaded
- Deprecating the [Remote Shader Compiler (RSC)]({% post_url /2019/2019-09-14-lmbr_build %}#remote-shader-compiler); shader permutations can be found via depdendency analysis rather than 

In this first release, the first two benefits are addressed.  Hopefully the capabilities enabled by this new feature will expand.

[Top-level userguide docs](https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-bundler-intro.html), especially the overview of ["What is the Amazon Lumberyard Asset Bundler?"](https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-bundler-overview.html).

https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-bundler-starter-game.html
https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-bundler-tutorial-simple.html

First, we'll create a release build for StarterGame similar to [SamplesProject]({% post_url /2019/2019-09-14-lmbr_build %}).

`lmbr_waf show_option_dialog` and from __Game Projects__ enable __Starter Game__ and disable the rest.

```powershell
# OPTIONAL: if you previously worked with something need to rerun asset builders to generate dependencies for asset bundler
Remove-Item -Recurse ./Cache
# OPTIONAL: delete previous build output
Remove-Item -Recurse ..\..\StarterGameRelease\,.\build\pc\StarterGame\,.\startergame_pc_paks\

./lmbr_waf configure
./lmbr_waf build_win_x64_vs2017_profile -p all

# Build release executable and add to build
./lmbr_waf build_win_x64_vs2017_release -p game_and_engine
New-Item -Type Directory ../../StarterGameRelease
Copy-Item -Recurse .\Bin64vc141.Release\ ../../StarterGameRelease/Release

# In another shell, start RemoteShaderCompiler
.\Tools\CrySCompileServer\x64\profile\CrySCompileServer

.\Build_AssetBundler_AuxiliaryContent_PC.bat --game StarterGame
Copy-Item -Recurse .\startergame_pc_paks\* ..\..\StarterGameRelease\

# Pack shaders and add to build
.\lmbr_pak_shaders.bat StarterGame D3D11 pc .\Tools\CrySCompileServer\x64\profile\Cache\StarterGame\PC-D3D11_FXC-D3D11\ShaderList_D3D11.txt
Copy-Item .\build\pc\StarterGame\*.pak ..\..\StarterGameRelease\startergame\

# It's safe to shutdown the RSC

# Build .assetlist file(s)
.\Bin64vc141\AssetBundlerBatch assetLists --addSeed .\startergame_pc_paks\startergame\levels\startergame\level.pak --seedListFile .\StarterGame\Levels\StarterGame\SeedAssetList.seed --assetListFile .\StarterGame\AssetListFiles\startergame.assetlist --addDefaultSeedListFiles --allowOverwrites


# Use .assetlist to build .pak bundle
# NB: AssetBundler currently doesn't work with `outputBundlePath` in parent of LY root
.\Bin64vc141\AssetBundlerBatch bundles --assetListFile .\StarterGame\AssetListFiles\startergame_pc.assetlist --outputBundlePath .\build\pc\startergame\Bundles\startergame.pak --allowOverwrites
Copy-Item -Recurse .\build\pc\startergame\Bundles\* ..\..\StarterGameRelease\startergame\

# Launch game specifying a map
StarterGameLauncher.exe +map startergame
```

https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-bundler-tutorial-simple.html
```powershell
# Create engine_pc.assetlist
Bin64vc141\AssetBundlerBatch.exe assetLists --addDefaultSeedListFiles --assetListFile engine.assetlist --allowOverwrites
# Create engine_pc.pak
Bin64vc141\AssetBundlerBatch.exe bundles --assetListFile engine_pc.assetlist --outputBundlePath engine.pak --allowOverwrites

# Create startergame.seed
Bin64vc141\AssetBundlerBatch.exe seeds --addSeed .\StarterGame\Levels\StarterGame\level.pak --addSeed .\StarterGame\project.json --addSeed .\StarterGame\gems.json --seedListFile startergame.seed --allowOverwrites
# Create startergame_pc.assetlist
Bin64vc141\AssetBundlerBatch.exe assetLists --seedListFile .\startergame.seed --assetListFile ./startergame.assetlist
# Create startergame_pc.pak
Bin64vc141\AssetBundlerBatch.exe bundles --assetListFile .\startergame_pc.assetlist --outputBundlePath startergame.pak

Set-Location ..\..\StarterGameRelease\StarterGame\
..\..\lumberyard\dev\Bin64vc141\AssetBundlerBatch.exe assetLists --addDefaultSeedListFiles --assetListFile engine.assetlist
```