---
layout: post
title: PowerShell Crash Course (Part 2)
tags:
- devops
- powershell
- scripting
series: pwsh_crash_course
---

I've remained committed to improving my powershell game.  Reaching for it instead of shoddy `bat` files.


## More

- [Enums](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_enum)
- [Pipeline functions](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions#piping-objects-to-functions)
- [Modules](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Splatting](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_splatting)
- [Data sections](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_data_sections)

## Remoting

https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ssh-remoting-in-powershell-core


- [About Functions Advanced Parameters](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters)


## Portable Scripts

- [Breaking changes in PowerShell Core 6](https://docs.microsoft.com/en-us/powershell/scripting/whats-new/breaking-changes-ps6)
- ["Tips for writing cross-platform powershell code"](https://powershell.org/2019/02/tips-for-writing-cross-platform-powershell-code/)


https://superuser.com/questions/690258/powershell-gci-filter-with-compact-output

[Measure-Command](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command)


https://devblogs.microsoft.com/scripting/maximizing-the-power-of-here-string-in-powershell-for-configuration-data/


Downloading files:
https://blog.jourdant.me/post/3-ways-to-download-files-with-powershell
While [BITS](https://docs.microsoft.com/en-us/windows/win32/bits/about-bits) via `Start-BitsTransfer` is a great option for Windows, `Invoke-WebRequest` works best for multi-platform scripts:
```powershell
Invoke-WebRequest https://sh.rustup.rs -OutFile rustup-init.sh
```

## &&

https://github.com/PowerShell/PowerShell/pull/9849
In [7.0.0-preview.5](https://github.com/PowerShell/PowerShell/milestone/71)
https://github.com/PowerShell/PowerShell/releases/tag/v7.0.0-preview.5

If you're using [Windows Terminal](https://github.com/microsoft/terminal), __V__ > __Settings__ (or `Ctrl+,`) to open `profiles.json`:
```json
{
    "...": "...",
    "profiles" : 
    [
        {
            "...": "...",
            "commandline" : "C:\\Program Files\\PowerShell\\7-preview\\pwsh.exe",
            "name" : "PowerShell Core",
            "...": "...",
        },
    ]
}
```

Opening a new "PowerShell Core" tab should display:
```
PowerShell 7.0.0-preview.5
Copyright (c) Microsoft Corporation. All rights reserved.
```

## Profiles

https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles
`$profile` displays current profile:
`$home\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`


`which` and `whereis` [SO](https://stackoverflow.com/questions/63805/equivalent-of-nix-which-command-in-powershell):
`Get-Command <command>`

## Aliases

[`Get-Alias`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-alias):
```powershell
# List all aliases
alias
Get-Alias
# Get alias for `Get-Command`
Get-Alias -Definition Get-Command
# Get aliases matching `gc*`
Get-Alias gc*
Get-Alias -Name gc*
```

## Substrings

`-like` not `-contains` which is for collections

```powershell
$string = "windows-2019"
$string -contains '*2019*' # False
$string -like '*2019*' # True
$string -like '2019' # False
$string.Contains('*2019*') # False
$string.Contains('2019') # True
```

## Existance

https://devblogs.microsoft.com/scripting/use-a-powershell-function-to-see-if-a-command-exists/

## Loops

- [`While`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_while)
- [`For`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_for)
- [`ForEach-Object`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)
- [`ForEach`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_foreach) statement
- [`Do`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_do) while/until

- [`Break`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_break)
- [`Continue`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_continue)

```powershell
Get-Alias -Definition "*ForEach*"

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           % -> ForEach-Object
Alias           foreach -> ForEach-Object
```

```powershell
0..10 | % {
    # Commands to repeat here
}
```

## More error handling

[`Trap`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_trap)
[`$ErrorActionPreference`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_preference_variables?ranMID=24542&ranEAID=je6NUbpObpQ&ranSiteID=je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg&epi=je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg&irgwc=1&OCID=AID2000142_aff_7593_1243925&tduid=(ir__dwqvlbyvjokfrnl0kk0sohz30f2xgldgb39s6tas00)(7593)(1243925)(je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg)()&irclickid=_dwqvlbyvjokfrnl0kk0sohz30f2xgldgb39s6tas00#erroractionpreference)
