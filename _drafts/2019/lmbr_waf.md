---
layout: post
title: Lumberyard and Waf
tags:
- lumberyard
- gamedev
- waf
canonical_url: 
series: Amazon Lumberyard
---


## Waf Basics

`Tools/build/waf-1.7.13/`

| File | Description
|-|-|
| `wscript` | 
| `*.waf_files` | 

Regenerate Visual Studio solution file in `dev/Solutions/`:
```powershell
./lmbr_waf.bat msvs
```


## Adding a File

Check the project's `_WAF_/wscript` for the list of files.  Here's `Code\CryEngine\CryCommon\wscript`:
```python
def build(bld):
    bld.CryFileContainer(
        # SNIP
        target             = 'CryCommon',
        vs_filter          = 'Common',
        file_list          = 'crycommon.waf_files',
        # SNIP
)
```

`.waf_files` is json specifying the files and VS solution filters:
```json
{
    "none":
    {
        "Root":
        [
            "QTangent.h"
        ],
        "Interfaces_h": [
            "InputTypes.h",
            "GameplayEventBus.h",
            "InputEventBus.h",
            "InputNotificationBus.h",
    ...
```

```powershell
./lmbr_waf.bat configure
```

Generates a `Common` folder with a "CryCommon" project containing `QTangent.h` in the root and `Interfaces_h` "folder":  
![](/assets/lmbr_vs_waf_files.png)

## Adding a Spec

Adding a spec:
https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-using-spec.html


https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-project-settings.html

## Adding a 3rd-Party Library

https://docs.aws.amazon.com/lumberyard/latest/userguide/waf-adding-third-party-libraries.html

## Tips

The configurations have rather long names, embiggen __Solution Configurations__ drop-down:
![](/assets/lmbr_vs_sln_config.png)

https://visualstudioextensions.vlasovstudio.com/2014/08/14/adjusting-the-width-of-solution-configurations-drop-down-list-in-the-visual-studio-toolbar/

If you're a licensed Playstation/Xbox developer you can get access to Lumberyard PS4/Xbone code from the [FAQ](https://aws.amazon.com/lumberyard/faq/#Q._How_do_I_get_started_with_Xbox_and_PlayStation_game_development.3F):

> If you are a licensed Microsoft Xbox developer, please e-mail your name, studio name, and the licensed e-mail address to lumberyard-consoles@amazon.com. If you are a licensed Sony PlayStation developer, please visit SCE DevNet. Under the Middleware Directory click "Confirm Status" for Amazon Lumberyard.