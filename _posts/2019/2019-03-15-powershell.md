---
layout: post
title: PowerShell Crash Course
tags:
- devops
- powershell
- scripting
- windows
---

Let's be honest, `cmd.exe` sucks.  I have fond memories of MS-DOS 6.0/6.22, but after getting over the learning curve of Linux/Unix it's tough to go back.  When PowerShell 1.0 was announced I was excited; finally a "real" CLI for Windows.  But I couldn't really be asked to learn it and kept installing [cygwin](https://www.cygwin.com/) or [msys](http://mingw.org/wiki/msys).

[After discovering]({% post_url /2018/2018-09-26-new-microsoft %}) [PowerShell Core](https://github.com/PowerShell/PowerShell) is multi-platform, it's back on my radar because of a few use cases:

- [Windows automation]({% post_url /2019/2019-02-04-vagrant %})
- CI/CD
    - Our [JenkinsFile](https://jenkins.io/doc/book/pipeline/) is bloated with inline script (some was moved to C#, but that initiative was unsuccessful)
    - [Some AppVeyor]({% post_url /2018/2018-09-21-dotnet %}#appveyor) where `pwsh:` [works on both Linux/Windows](https://www.appveyor.com/docs/getting-started-with-appveyor-for-linux/#running-windows-and-linux-builds-side-by-side)
- Windows OEM
    - If you customize/automate Windows OS setup as a Windows OEM licensee, device OEM/ODM, or enterprise IT you pretty much have to use PowerShell

This is more in the vain of cheat-sheet/quick reference/cookbook than tutorial.  If you're not comfortable picking up new (scripting) languages, this may be unhelpful.

## Shell Keyboard Shortcuts

Display list of all shortcuts:
```powershell
Get-PSReadlineKeyHandler
```

On macOS/Linux it defaults to `emacs` "edit mode".  If you've used emacs (or [bash's emacs mode](https://www.gnu.org/software/bash/manual/html_node/Command-Line-Editing.html)) you'll feel right at home.  On Windows, the default is `Windows` but you can:
```powershell
Set-PSReadLineOption -EditMode Emacs
```

If you're new to PowerShell, I highly recommend switching to emacs mode.  If for no other reason you'll also familiarize yourself with Bash- should you ever find yourself at a Linux terminal.

Commands for every day use:
```powershell
# Movement
Ctrl+a # Beginning of line
Ctrl+e # End of line
Ctrl+f # Forward one character
Ctrl+b # Back one character
Alt+f # Forward one word
Alt+b # Back one word

# Editing
Alt+. # Insert last argument of previous command
Ctrl+d # Delete character
Alt+d # Delete word
Ctrl+u # Delete to beginning of line
Ctrl+k # Delete to end of line

# Command History
Ctrl+p # Previous command
Ctrl+n # Next command
Ctrl+o # Execute command and advance to next
Ctrl+r <text> # Search command history for <text>
```

`Alt+.` is one I wish I had learned the first day I installed Linux.  Tip: you can press it repeatedly to cycle through the history of last arguments.

Same with `Ctrl+o`.  If you need to redo a sequence of commands: `Ctrl+p` back to the start, `Ctrl+o` `Ctrl+o`... throw in `Ctrl+n` if you need to skip one, etc.

If you're on macOS/OSX and using the default terminal `Alt` is `Esc`.  Or, you can use `Option` (recommended) via __Terminal > Preferences__:  
![](/assets/osx_term_option.png)


## Basic Syntax

Literals and variables:
```powershell
$boolean = $true # or `$false`
$string = "string"
$int = 42
$array = 1, 2, 3
$array2 = @(1, 2, 3)
$array[0] = $null # Remove first item
$hash = @{first = 1
    "second" = 2; third = 3
}
# Add to hashtable
$hash += @{4 = "fourth"}
$hash["fifth"] = 5 # Key has to be quoted here
$hash[4] = $null # Remove value (but not key) from hash

# List all environment variables
dir env:
# String with `PATH` environment variable
"$env:PATH ${env:PATH}: safer"
# Multi-line "here string"
@"
"Here-string" with value $env:PATH
"@
# Escape character
"literal `$ or `" within double-quotes"
# Evaluate expression
"Hello $(echo world)"

# Casting locks variable type
[int[]]$ints = "1", "2", "3"
$ints = "string" # Throws exception
# Destructuring
$first, $rest = $ints # first = 1; $rest = 2,3
```

Control-flow:
```powershell
$value = 42
if ($value -eq 0) {
    # Code
} elseif ($value -gt 1) {
} else {
}

$value = "value"
# Match against each string/int/variable/expression case
switch ($value)
{
    "x" { echo "matched string" }
    1 { echo "matched int" }
    $var { echo "matched variable" }
    { $_ -gt 42 }{ echo "matched expression" }
    default { }
}
$collection = 1,2,3,4
# Matched against each element of collection.  `$_` is current item.  `Break` applies to entire collection
switch ($collection)
{
    1 { echo $_ 1 }
    { $_ -gt 1 } { echo "$_ Greater than 1" }
    3 { echo $_ 3; break }
}
# Output is (NB: there's no 4):
#1 1
#2 Greater than 1
#3 Greater than 1
#3 3

# Loops
foreach ($val in $collection) {
}

while ($value -gt 0) {
    $value--
}
```

Cheat sheet:  

- __Assignment__: `+=` `-=` `*=` `/=` `++` `--` (e.g. `++$int` or `$int++` or `$int += 1`)
- __Equality__: `-eq` `-ne` `-gt` `-ge` `-lt` `-le`
- __Matching__: `-like` `-notlike` (wildcard), `-match` `-notmatch` (regex; `$matches` contains matching strings)
- __Containment__: `-contains` `-notcontains` `-in` `-notin`
- __Type__: `-is` `-isnot`
- __Logic__: `-and` `-or` `-xor` `-not` or `!` (e.g. `$a -and $b` or `-not $a` or `!$a`)
- __Replacement__: `-replace` (replaces a string pattern)
- Other than the last, all return `$true` or `$false`
- All are case-insensitive.  For case-sensitive prefix with `c` (e.g. `-clike`)
- If input is collection, output is a collection of matches

More info:  

- [Comparison operators](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_comparison_operators)
- [`While`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_while)
- [`For`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_for)
- [`ForEach-Object`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/foreach-object)
- [`ForEach`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_foreach) statement
- [`Do`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_do) while/until
- [`Break`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_break)
- [`Continue`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_continue)

## Essentials

```powershell
# List commands containing "Path"
Get-Command -Name *path*

# Get help for `Get-Command`
Get-Help Get-Command

# List properties/methods of object
Get-Command | Get-Member

# Change directory
cd output/Debug
Set-Location output/Debug
# pushd/popd
pushd tmp/
popd
Push-Location tmp/
Pop-Location
cd - # Go back to previous directory

# Current file/module's directory
$PSScriptRoot

ls
dir # also works which is freaky/helpful for migration
Get-ChildItem
# Pattern glob
ls *.jpg
Get-ChildItem *.jpg
# Just files
Get-ChildItem -File
# Just directories
Get-ChildItem -Directory
Get-ChildItem | ForEach-Object { $_.Name }
Get-ChildItem | Where-Object {$_.Length -gt 1024}
# find: `-Force` includes "hidden" folders.  `-Path` and `-Include` also accept comma-separated list.
Get-Childitem –Path C:\ -Include *HSG* -Exclude *.JPG,*.MP3,*.TMP -File -Recurse -ErrorAction SilentlyContinue -Force

# Create directory
md tmp/
New-Item -ItemType Directory -Name tmp/ -Force | Out-Null
# touch; create file
New-Item -ItemType file filename
(gci filename).LastWriteTime = Get-Date # Update timestamp

# Copy file to existing folder
Copy-Item filename tmp/
# Copy folder to new folder (create tmp1/filename)
Copy-Item tmp/ tmp1/ -Recurse
# Copy folder to existing folder (creates tmp1/tmp/filename)
Copy-Item tmp/ tmp1/ -Recurse

# Move files/folders
mv old new
Move-Item -Path old -Destination new
Move-Item old new

# cat
Get-Content filename
# tail -f
Get-Content filename -Wait
Get-Content filename -Tail 1 -Wait # Get last line and wait

# Remove directory
Remove-Item tmp1/ -Recurse
# Remove files matching pattern
Remove-Item tmp/* -Include *.txt -Exclude *.keep.txt
```

- [Get-ChildItem](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [New-Item](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [Copy-Item](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/copy-item)
- [Move-Item](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/move-item)
- [Get-Content](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Remove-Item](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/remove-item)

```powershell
# Add to PATH
$env:PATH += ";$env:USERPROFILE" # `;` for Windows, `:` for *nix
$env:PATH += [IO.Path]::PathSeparator + $(pwd) # Any platform

# Check environment variable `GITHUB_TOKEN` is set
Test-Path Env:\GITHUB_TOKEN
# Test for file/directory
Test-Path subdir/child -PathType Leaf # `Container` for directory
# Get all environment variables
Get-ChildItem Env:

# Write to stdout, redirect stderr to stdout, send stdout to /dev/null
Write-Output "echo" 2>&1 > $null
&{
    Write-Warning "warning"
    Write-Output "stdout"
# Append warnings to tmp.txt, rest to /dev/null
} 3>> ./tmp.txt | Out-Null
# Write to stderr, redirect all, append to file
Write-Warning "oops" *>> ./tmp.txt

# Execute string
$ls = "ls"
& $ls
& $ls -l # with args
# Execute string with args
$ls_l = "ls -l"
Invoke-Expression $ls_l
# Execute file `script.ps1`
& ./script
$file = "./script.ps1"
& $file

# Subexpression; string contains result of command
"Result: $(Test-Path Env:\GITHUB_TOKEN)"

# Execute command looking for failure text
$res = Invoke-Expression "& $cmd 2>&1"
if ($LASTEXITCODE -and ($res -match "0x800700C1")) {
    # Do something
}
```

- [Automatic variables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables)
- [Providers](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_providers) (like `Env:\`)
- [Redirection](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_redirection)
- [Operators](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_operators) (like [call operator `&`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_operators#call-operator-) and [subexpression operator `$()`](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_operators#subexpression-operator--))
- [Blog on searching](https://devblogs.microsoft.com/scripting/use-windows-powershell-to-search-for-files/)

## Error Handling

Powershell has terminating (i.e. exceptions) and non-terminating errors.
```powershell
# Delete PathToDelete/ folder recursively ignoring all errors
Remove-Item -Force -Recurse -ErrorAction Ignore PathToDelete

# Make terminating error
Write-Error "fail" -ErrorAction Stop
throw "fail"

# Non-terminating errors are terminating
$ErrorActionPreference = "Stop"

# Handle terminating error
try {
    throw "fail"
} catch [System.Management.Automation.RuntimeException] {
    Write-Output "Throw'd: $_"
] catch [Microsoft.PowerShell.Commands.WriteErrorException] {
    Write-Output "Write-Error'd"
} catch {
    # Any error
} finally {
    # Always executes
}

# Handling non-terminating errors
if ($LastExitCode > 0) {
    # Exit code of last program >0, which might mean it failed
}
if ($?) {
    # Last operation succeeded
} else {
    # Last operation failed
}
```

- [Common parameters](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters) (like `-ErrorAction`)
- [Preference variables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_preference_variables) (like `$ErrorActionPreference`)

## Parameters and Functions

Command-line arguments to a script are handled as `param()` placed at the top of file.

```powershell
function Hello { echo Hi }
# Call the function
Hello
# Output: Hi

# Function with two named params.  First with type and default (both optional)
function HelloWithParams {
    param([string]$name = "<unknown>", $greeting)
    echo "hello $name! $greeting"
}
HelloWithParams 1 2
# Output: hello 1! 2
HelloWithParams -greeting 40
# Output: hello <unknown>! 40

# Function with switch and positional parameters
function Greeting {
    param([switch]$flag)
    echo "hello $flag $args"
}
Greeting more stuff
# Output: hello False more stuff
Greeting -flag more stuff
# Output: hello True more stuff
Greeting -flag:$false more stuff
# Output: hello False more stuff
```

```powershell
function PositionalParams {
    param(
        [parameter(Position=0)]
        $greeting,
        [string]$name,
        [parameter(Position=1)]
        $tail
        )
    echo "$greeting $name$tail"
}
PositionalParams hi "!"
# Output: hi  !
PositionalParams hi -name jake "!"
# Output: hi jake!
PositionalParams hi "!" -name jake
# Output: hi jake!

function FormalHello {
    param(
        # Params default to optional
        [parameter(Mandatory=$true, HelpMessage="Initial greeting")]
        [string]$greeting,
        # If have multiple values have to use @()
        [string[]]$name = @("Sir", "<unknown>")
        )
    echo "$greeting $name $suffix"
}
FormalHello "Greetings"
# Output: Greetings Sir <unknown>
FormalHello -name sir,jake,3rd -greeting welcome
# Output: welcome sir jake 3rd
```


## Misc

__Visual Studio Code__:  
- Use [the extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell).
- On Windows, install PowerShell Core and in Visual Studio Code click the "PowerShell Session Menu":  
    ![](/assets/vscode_pwsh_switch.png)

    From command pallete that appears pick __Switch to PowerShell Core 6 (x64)__.

__Jenkins__:
- Inline powershell or call a script:
    ```groovy
    powershell '''
        & ./script.ps1
        '''
    ```
- Call script and handle exit code:
    ```groovy
    def res = powershell returnStatus: true, script: '''
        & ./script.ps1
        '''
    if (res != 0) {
        currentBuild.result = 'UNSTABLE'
    }
    ```