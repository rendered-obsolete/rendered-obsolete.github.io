---
layout: post
title: PowerShell Crash Course (Part 3)
tags:
- devops
- powershell
- scripting
series: pwsh_crash_course
---


https://devblogs.microsoft.com/scripting/easily-compare-two-folders-by-using-powershell/

https://docs.microsoft.com/en-gb/powershell/module/microsoft.powershell.core/about/about_object_creation
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest
https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod

## Working with XML

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<paramsfo>
 <param key="APP_TYPE">0</param>
 <param key="VERSION">01.00</param>
 <!-- Lots more -->
</paramsfo>
```

We need to do things like find elements with particular attributes and set values.

If you use C#, you probably immediately think of [LINQ](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/).  Turns out you [can use LINQ in Powershell](https://powershell.org/2018/05/100887/) but it's a bit unwieldy.

For simple cases like this, you can use [XPath](https://docs.microsoft.com/en-us/dotnet/standard/data/xml/select-nodes-using-xpath-navigation) (from [this SO](https://stackoverflow.com/questions/31244005/modify-xml-from-the-command-line)):
```powershell
# Load file as XmlDocument
$xml = [xml](Get-Content -Path $path)
# Find <param> child element with `key='VERSION'` attribute
$node = $xml.SelectSingleNode("//paramsfo/param[@key='VERSION']")
# Replace `01.00` with `02.01`
$node.InnerText = "02.01"
# Write changes to disk
$xml.Save($path)
```

Short and sweet.  `[xml]` is short-hand for [`[System.Xml.XmlDocument]`](https://docs.microsoft.com/en-us/dotnet/api/system.xml.xmldocument) and gives you access to the full C# XML DOM.

More:

- [Simple guide to xpath syntax](https://www.w3schools.com/xml/xpath_syntax.asp)
- [C# LINQ and corresponding Powershell](https://www.red-gate.com/simple-talk/dotnet/net-framework/high-performance-powershell-linq/)

## More

- [Enums](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_enum)
- [Pipeline functions](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions#piping-objects-to-functions)
- [Modules](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_modules)
- [Splatting](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_splatting)
- [Data sections](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_data_sections)

- [Advanced Function Parameters](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters)

## More error handling

[`Trap`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_trap)
[`$ErrorActionPreference`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_preference_variables?ranMID=24542&ranEAID=je6NUbpObpQ&ranSiteID=je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg&epi=je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg&irgwc=1&OCID=AID2000142_aff_7593_1243925&tduid=(ir__dwqvlbyvjokfrnl0kk0sohz30f2xgldgb39s6tas00)(7593)(1243925)(je6NUbpObpQ-e8ALqi5qBUN7mFO9gJmCUg)()&irclickid=_dwqvlbyvjokfrnl0kk0sohz30f2xgldgb39s6tas00#erroractionpreference)


## Processes

[Get-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
[Stop-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/stop-process)
[Wait-Process](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/wait-process)

https://docs.microsoft.com/en-us/powershell/scripting/samples/managing-processes-with-process-cmdlets

## Jobs

```powershell
# Start job in background (sleeps for 200 seconds)
$job = Start-Job { param($secs) Start-Sleep $secs } -ArgumentList 200
# Or
$job = Start-Sleep 200 &
# Wait for it with a timeout
Wait-Job $job -Timeout 4

# Jobs run in their own session, use -ArgumentList
$value = "hi"
Start-Job { Write-Output "value=$value" } | Wait-Job | Receive-Job
# Output: value=
Start-Job { Write-Output "value=$args" } -ArgumentList $value | Wait-Job | Receive-Job
# Output: value=hi


# Start a bunch of work in parallel
Get-Job | Remove-Job # Remove existing jobs
$MaxThreads = 2 # Limit concurrency
foreach ($_ in 0..10) {
    # Wait for one of the jobs to finish
    while ($(Get-Job -State Running).count -ge $MaxThreads) {
        Start-Sleep 1
    }
    Start-Job -ScriptBlock { Start-Sleep 2 } # Random work
}
# Wait for them all to finish
while ($(Get-Job -State Running)){
    Start-Sleep 1
}
```

- [Jobs](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_jobs)
- Be careful if you use relative paths: jobs start from `$HOME` on macOS/Linux and `Documents/` on Windows
- Need `Receive-Job` to see stdout/stderr
- Parallel work snippet from [this SO](https://stackoverflow.com/questions/43685522/running-tasks-parallel-in-powershell)


## WSL

`wsl` enter wsl shell.

## Measure

http://woshub.com/powershell-get-folder-sizes/#:~:text=To%20get%20the%20size%20of,files%20and%20directories%20in%20PowerShell.

## Portable Scripts

- [Breaking changes in PowerShell Core 6](https://docs.microsoft.com/en-us/powershell/scripting/whats-new/breaking-changes-ps6)
- ["Tips for writing cross-platform powershell code"](https://powershell.org/2019/02/tips-for-writing-cross-platform-powershell-code/)



Sometimes you just can't avoid platform-specific branches.  Powershell Core provides [automatic variables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables) `$IsLinux` and `$IsMacOS` to determine the host OS.  However, the same is not true on Windows for "legacy" Powershell (non-Core):
```powershell
# $IsWindows is in Core, $PSVersionTable is in vanilla Powershell
$is_windows = $IsWindows -or $PSVersionTable.PSEdition -eq "Desktop"
```

To [determine if you're running on Powershell Core](https://devblogs.microsoft.com/scripting/powertip-identify-if-you-are-running-on-powershell-core/):
```powershell
# Returns `"Core"` on Powershell Core
$PSVersionTable.PSEdition
```
