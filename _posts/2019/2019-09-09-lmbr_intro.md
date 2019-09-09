---
title: Amazon Lumberyard Intro
tags:
- lumberyard
- gamedev
canonical_url: 
series: lumberyard
---

Some years ago we worked with [CryEngine](https://www.cryengine.com), and more recently we've started working with its descendant- [Amazon Lumberyard](https://aws.amazon.com/lumberyard/).  Both game engines originated from the same codebase, but since Amazon acquired the source code in 2015 they have diverged significantly.

Here we'll look at doing the basics- getting it building and running.  If you're of the creative/artistic persuasion and more interested in white-boxing levels or massaging materials, check out some of the many excellent resources on their respective dev portals, YouTube, etc.

## Git Started

First, make sure you appease the [Lumberyard hardware requirements](https://docs.aws.amazon.com/lumberyard/latest/userguide/setting-up-system-requirements.html#required-hardware-for-lumberyard):

    3GHz minimum quad-core processor
    8 GB RAM minimum
    2 GB minimum DirectX 11 or later compatible video card
    NVIDIA driver version 368.81 or AMD driver version 16.15.2211 graphics card
    60 GB minimum of free disk space

    To compile builds:
    14 GB RAM minimum

Besides [downloading the installer](https://aws.amazon.com/lumberyard/) (which also includes the source code), you can pull it from github:
```powershell
git clone https://github.com/aws/lumberyard
cd lumberyard
./git_bootstrap.exe
```

If `git_bootstrap` doesn't work for some reason (it didn't for me):
1. Download the zip specified in `bootstrap_config.json` by some other means
1. Unzip to the repository root
1. Run `./SetupAssistant.bat`

__IMPORTANT__: You need a lot of disk space.  Around 30GB for source code, required SDKs, and the samples.  Another 50GB to build one configuration of the game, editor, and tools.

## Configuration

Lumberyard uses [WAF](https://waf.io/) for it's build system via `dev/lmbr_waf.bat`.

The [user guide contains a list of commands](https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-commands.html), but to get started you'll need:

```powershell
# Detailed commands and options
./lmbr_waf.bat --help
# Just list of commands
./lmbr_waf.bat --list
# GUI configuration
./lmbr_waf.bat show_option_dialog
```

`show_option_dialog` presents a bunch of different options, but perhaps the most important are __Visual Studio Project Generator__ and __Game Projects__:

| VS Project Gen | Game Projects |
|-|-|
| ![](/assets/lmbr_waf_vs_project_gen.png) | ![](/assets/lmbr_waf_game_projects.png) |

If you don't want to futz with the janky Tkinter UI, you can also manually edit `dev/_WAF_/user_settings.options` and then run `lmbr_waf configure`.


## Building

Waf commands follow the format of `(build|clean|package|deploy)_platform[_arch]_toolchain_configuration`.  For example:
```powershell
./lmbr_waf.bat build_win_x64_vs2017_profile
```

__NB__:  If you don't have [Incredibuild](https://www.incredibuild.com) you're looking at hefty build time; 35 minutes on 12-core i9/64 GB/SSD.

"Specs" are pre-defined subsets of projects: "all" is everything, "game", "tools", etc.  They are defined in `dev/_WAF_/specs/` and appear in Visual Studio as __Solution Configurations__.  You can build a spec from the command line:
```powershell
./lmbr_waf.bat <command> -p <spec>
# Build engine and editor
./lmbr_waf.bat build_win_x64_vs2017_profile -p engine_and_editor
# Build game(s) and engine
./lmbr_waf.bat build_win_x64_vs2017_profile -p game_and_engine
```

### Common Problems

If you're using the latest version of Visual Studio 2017, waf may complain that [it can't find Visual Studio](https://forums.awsgametech.com/t/visual-studio-2017-compile-problems/6794).  In `show_option_dialog`, change __Windows Options > win_vs2017_vswhere_args__ to:
```
-version [15.7.27703.2035,16]
```

Alternatively, in `dev/_WAF_/usersettings.options`:
```ini
# OLD
;win_vs2017_vswhere_args = -version [15.7.27703.2035,15.9.28307.665]
# NEW
win_vs2017_vswhere_args = -version [15.7.27703.2035,16]
```

__IMPORTANT__: Do _not_ insert a space in the version array (i.e. `[15.7.27703.2035, 16]`).  It will fail to find a Visual Studio install.  Lines 795-796 in `dev\Tools\build\waf-1.7.13\lmbrwaflib\mscv_helper.py` are to blame.


If it fails with the innocuous looking `unable to find QT`:
```
[SETTINGS] msvs_version = 15 (default 14)
[SETTINGS] win_vs2017_vswhere_args = -version [15.7.27703.2035, 16] (default -version [15.7.27703.2035,15.9.28307.665])                                                            [SETTINGS] specs_to_include_in_project_generation = all,game_and_engine,engine_and_editor,game (default all, game, game_and_engine)
[SETTINGS] Target Output folder (win_x64_vs2017/profile): Bin64vc141
[WARNING] Incredibuild disabled by build option
unable to find QT
unable to find QT
Traceback (most recent call last):
<SNIP>
  File "Tools\build\waf-1.7.13\lmbrwaflib\cry_utils.py", line 872, in add_compiler_dependency
    if os.path.isabs( self.env['CC'] ):
  File "D:\projects\lumberyard\dev\Tools\Python\2.7.12\windows\lib\ntpath.py", line 59, in isabs
    return s != '' and s[:1] in '/\\'
TypeError: 'in <string>' requires string as left operand, not list
```

It could be any of several issues:
1. Missing one or more [Visual Studio requirements](https://docs.aws.amazon.com/lumberyard/latest/userguide/setting-up-system-requirements.html#required-developer-tools-for-lumberyard)
1. MFC may also be required (forum post [#1](https://forums.awsgametech.com/t/building-project-failed-error-executing-waf/4868), [#2](https://forums.awsgametech.com/t/visual-studio-2017-compile-problems/6794))
1. Problem detecting Visual Studio, check output of `lmbr_waf configure` for clues

If you get a bunch of the following errors, try re-running `lmbr_waf configure`:
```
ObjectStream.cpp
XXX\dev\Code\Framework\AzCore\AzCore/XML/rapidxml.h(20): fatal error C1083: Cannot open include file: 'rapidxml/rapidxml.h': No such file or directory
compression.cpp
XXX\dev\Code\Framework\AzCore\AzCore\Compression\compression.cpp(24): fatal error C1083: Cannot open include file: 'zlib.h': No such file or directory
ScriptContextDebug.cpp
XXX\dev\Code\Framework\AzCore\AzCore/Script/lua/lua.h(16): fatal error C1083: Cannot open include file: 'Lua/lua.h': No such file or directory
ScriptPropertyTable.cpp
XXX\dev\Code\Framework\AzCore\AzCore\Script\ScriptPropertyTable.cpp(20): fatal error C1083: Cannot open include file: 'Lua/lualib.h': No such file or directory
```

If you get an error like:
```
[2585/8092] pch_msvc (win_x64_vs2017|profile): Code\Tools\AzCodeGenerator\AZCodeGenerator\precompiled.cpp -> BinTemp\win_x64_vs2017_profile\Code\Tools\AzCodeGenerator\AZCodeGenerator\precompiled.4870660.obj BinTemp\win_x64_vs2017_profile\Code\Tools\AzCodeGenerator\AZCodeGenerator\precompiled.4870660.pch
precompiled.cpp
XXX\dev\code\tools\azcodegenerator\azcodegenerator\precompiled.h(31): fatal error C1083: Cannot open include file: 'clang/AST/Stmt.h': No such file or directory
```

It's probably an issue with an optional SDK:
![](/assets/lmbr_setup_clang.png)

Either install it, or build a different spec like `-p engine_and_editor`.

## Running

If you build an editor spec you can run `dev\Bin64vc141\Editor.exe` to launch the level editor:  
![](/assets/lmbr_editor.png)

Depending on the game projects you enabled, if you build a game spec you can run one or more `dev\Bin64vc141\<game>Launcher.exe`.

The first time you launch either you should see a notification about the [Asset Processor](https://docs.aws.amazon.com/lumberyard/latest/userguide/asset-pipeline-processor.html):  
![](/assets/lmbr_assetproc_notify.png)

The asset processor is responsible for preparing assets for use by the engine runtime.  This includes: processing [FBX](https://en.wikipedia.org/wiki/FBX) files, [DXT](https://en.wikipedia.org/wiki/S3_Texture_Compression)/[ETC](https://en.wikipedia.org/wiki/Ericsson_Texture_Compression) texture compression, and more.  We'll look at this more when we investigate the asset pipeline and release packaging.

This is StarterGameLauncher.exe:  
![](/assets/lmbr_starter_game.png)

Don't be surprised if it doesn't immediately look like this.  The sixth line of the debug text in the upper-right corner displays information about the "ShaderCache".  In this case:
- 22 "GCM" stands for _Global Cache Misses_ (i.e. missing shaders)
- 6 outstanding Asynchronous Requests
- Compiling is enabled

Again, we'll cover this when we look at the asset pipeline, but it is all about the [Remote Shader Compiler](https://docs.aws.amazon.com/lumberyard/latest/userguide/mat-shaders-custom-dev-remote-compiler.html) which compiles shaders for the game client.  If you wait a minute or two things will display correctly.  If it's not working, run `.\Tools\CrySCompileServer\x64\profile\CrySCompileServer_vc141x64.exe` (enabled with "all" spec).
