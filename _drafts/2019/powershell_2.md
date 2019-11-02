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

[`$()` "subexpression operator"][operators]


## More

- [Enums](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_enum)
- [Pipeline functions](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions#piping-objects-to-functions)
- [Modules](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Splatting](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_splatting)
- [Data sections](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_data_sections)
- [Operators][operators]
- [Advanced Function Parameters](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters)


https://superuser.com/questions/690258/powershell-gci-filter-with-compact-output

## time/wc

[Measure-Command](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command)

[Measure-Object](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-object)

https://devblogs.microsoft.com/scripting/maximizing-the-power-of-here-string-in-powershell-for-configuration-data/


## &&

https://github.com/PowerShell/PowerShell/pull/9849
In [7.0.0-preview.5](https://github.com/PowerShell/PowerShell/milestone/71)
https://github.com/PowerShell/PowerShell/releases/tag/v7.0.0-preview.5

If you're using [Windows Terminal](https://github.com/microsoft/terminal), [preview 1910](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1910-release/) should auto-detect it.  For older releases, __V__ > __Settings__ (or `Ctrl+,`) to open `profiles.json`:
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

For _Visual Studio Code_, __View__ > __Command Palette__ (`Ctrl+Shift+P`) and type `select default shell`.  It should present a drop-down list of options including the version you just installed.  You may need to restart VS Code to change the current shell.

Opening a new terminal should now display:
```
PowerShell 7.0.0-preview.5
Copyright (c) Microsoft Corporation. All rights reserved.
```

## Existence

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


## Formatting, awk, substrings

`-like` not `-contains` which is for collections

```powershell
$string = "windows-2019"
$string -contains '*2019*' # False
$string -like '*2019*' # True
$string -like '2019' # False
$string.Contains('*2019*') # False
$string.Contains('2019') # True
```


[Format-*](https://docs.microsoft.com/en-us/powershell/scripting/samples/using-format-commands-to-change-output-view)

[`-Split` "operator"](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_split)

[`-f` "format operator"][operators]

[ConvertTo-Json](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)


## grep

[Select-String](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)


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

## Profiles

https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles
`$profile` displays current profile:
`$home\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`


`which` and `whereis` [SO](https://stackoverflow.com/questions/63805/equivalent-of-nix-which-command-in-powershell):
`Get-Command <command>`


## Visual Studio

A recent VS 2019 update came "Developer PowerShell for VS 2019":  
![](/assets/vs2019_dev_pwsh_props.png)

```powershell
Import-Module "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\Common7\Tools\Microsoft.VisualStudio.DevShell.dll"
Enter-VsDevShell 20a49f0c
```

```powershell
PS> Get-Help Enter-VsDevShell

NAME
    Enter-VsDevShell

SYNTAX
    Enter-VsDevShell -VsInstallPath <string> [-SkipExistingEnvironmentVariables] [-StartInPath <string>] [-DevCmdArguments <string>] [-DevCmdDebugLevel {None | Basic | Detailed       | Trace}] [-SkipAutomaticLocation] [-SetDefaultWindowTitle] [<CommonParameters>]

    Enter-VsDevShell [-VsInstanceId] <string> [-SkipExistingEnvironmentVariables] [-StartInPath <string>] [-DevCmdArguments <string>] [-DevCmdDebugLevel {None | Basic | Detailed      | Trace}] [-SkipAutomaticLocation] [-SetDefaultWindowTitle] [<CommonParameters>]

    Enter-VsDevShell [-Test] [-DevCmdDebugLevel {None | Basic | Detailed | Trace}] [<CommonParameters>]
```



```powershell
Enter-VsDevShell -VsInstallPath "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\"
```

## Processes

[Get-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
[Stop-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/stop-process)
[Wait-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/wait-process)

https://docs.microsoft.com/en-us/powershell/scripting/samples/managing-processes-with-process-cmdlets

## Remoting

https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ssh-remoting-in-powershell-core



Downloading files:
https://blog.jourdant.me/post/3-ways-to-download-files-with-powershell
While [BITS](https://docs.microsoft.com/en-us/windows/win32/bits/about-bits) via `Start-BitsTransfer` is a great option for Windows, `Invoke-WebRequest` works best for multi-platform scripts:
```powershell
Invoke-WebRequest https://sh.rustup.rs -OutFile rustup-init.sh
```

## Portable Scripts

- [Breaking changes in PowerShell Core 6](https://docs.microsoft.com/en-us/powershell/scripting/whats-new/breaking-changes-ps6)
- ["Tips for writing cross-platform powershell code"](https://powershell.org/2019/02/tips-for-writing-cross-platform-powershell-code/)



[operators]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_operators