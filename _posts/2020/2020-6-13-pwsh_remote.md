---
layout: post
title: PowerShell Crash Course- Remoting
tags:
- devops
- powershell
- scripting
series: pwsh_crash_course
---

PowerShell "Remoting" enables you to create a shell or execute commands on a remote machine.  Basically, it's the PowerShell equivalent of the `ssh` command.  There are certain aspects of it that feel rather well-designed and almost novel that make it well worth learning.  Especially if your environment is heavily Windows-based.

# Setup for Windows Host/Client

PowerShell, being "native" to Windows, is relatively easy on Windows 10.  The [requirements](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements) basically boil down to running [Enable-PSRemoting](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting) in an admin powershell:
```powershell
Enable-PSRemoting -Force
```

# Setup for Linux/Mac Clients

[PowerShell Core](https://github.com/PowerShell/Powershell) is available for multiple platforms including Linux and Mac.  But PowerShell Remoting from a Linux/Mac client to a Windows host needs to use SSH instead of WinRM.  This requires:

1. Install SSHD (OpenSSH daemon) on Windows Host
1. Test Powershell Remoting
1. Optional Configuration
1. Copy Client Key to Host

Most of this comes from official docs on [OpenSSH installation](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse) and [SSH remoting](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ssh-remoting-in-powershell-core).

## Install SSHD on Windows Host

Everything here needs to be done as __admin__.

```powershell
Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'
# Install listed `OpenSSH.Server~~~~X.X.X.X`.  E.g.:
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Depending on your environment you may get:
`Add-WindowsCapability failed. Error code = 0x800f0954`.  If so, install it manually:

1. Download [latest Win32-OpenSSH release](https://github.com/PowerShell/Win32-OpenSSH/releases)
1. Run `install-sshd.ps1`

Edit `$env:ProgramData\ssh\sshd_config`:
```bash
# Uncomment the following
PasswordAuthentication yes
PubkeyAuthentication yes

# Add the following
Subsystem powershell c:/progra~1/powershell/7/pwsh.exe -sshs -NoLogo -NoProfile
```

Note the use of 8.3 path to avoid spaces on account of [this issue](https://github.com/PowerShell/Win32-OpenSSH/issues/784).  To verify the 8.3 name:
```powershell
Get-CimInstance Win32_Directory -Filter 'Name="C:\\Program Files"' |
  Select-Object EightDotThreeFileName
```

Finally, restart sshd:
```powershell
Restart-Service sshd
```

## Test Powershell Remoting

From Mac/Linux Powershell:
```powershell
Enter-PSSession WindowsHostnameOrIP
```
If it fails with:
```
Enter-PSSession: This parameter set requires WSMan, and no supported WSMan client library was found. WSMan is either not installed or unavailable for this system.
```
Explicitly use SSH or try the [workaround of installing an older version of OpenSSL](https://github.com/PowerShell/PowerShell/issues/11159):
```powershell
Enter-PSSession WindowsHostnameOrIP -SSHTransport
```

It also works from sh/bash/zsh/et al. but this probably results in `cmd.exe` shell on the remote Windows host:
```sh
ssh WindowsUsername@WindowsHostnameOrIP
# Or, if client/host usernames are the same
ssh WindowsHostnameOrIP
```

## Optional Configuration

When you ssh into a Windows host the default shell is `cmd.exe` _\*shudder\*_.  To change the default shell to powershell [configure SSHD on the Windows host](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration#configuring-the-default-shell-for-openssh-in-windows):
```powershell
# Again, 8.3 path to executable
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "c:/progra~1/powershell/7/pwsh.exe" -PropertyType String -Force
```

To avoid having to specify `WindowsUsername@...` with ssh, you can set a per-host default in `~/.ssh/config` on Linux/Mac:
```
Host WindowsHostnameOrIP
    HostName WindowsHostnameOrIP
    User WindowsUsername
```

## Copy Client Public Key to Host

Techinically this is optional, but key-based authentication is too convenient to overlook.  We assume you've already generated a key-pair on the client with `ssh-keygen`.  You can then follow the [PowerShell docs to deploy the key](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement#deploying-the-public-key).

Copy client public key to host (NB: use single-quotes to avoid local shell expansion of variables, and replace `id_rsa.pub` with name of your public key):
```sh
# If Windows host doesn't already have `authorized_keys` file
scp $HOME/.ssh/id_rsa.pub WindowsHostnameOrIP:"C:\Users\WindowsUsername\.ssh\authorized_keys"
# OR, if default Windows SSH shell is Powershell, you can append to the existing file or create it:
ssh WindowsHostnameOrIP '$Input | Add-Content $env:Home/.ssh/authorized_keys' < $HOME/.ssh/id_rsa.pub

# For completeness; for Linux/Mac hosts:
ssh-copy-id HostnameOrIP
# OR
ssh HostnameOrIP 'cat >> $HOME/.ssh/authorized_keys' < $HOME/.ssh/id_rsa.pub
```

In the above, `$Input` is an [automatic variable](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_automatic_variables#input) containing stdin (from `< $HOME/.ssh/id_rsa.pub`).

Test you can connect from Linux/Mac client to Windows host without inputting password:
```powershell
# In Linux/Mac PowerShell
Enter-PSSession WindowsHostnameOrIP -SSHTransport
# OR
ssh WindowsHostnameOrIP
```

# Commands

There's several good overview documents on remoting commands:

- [Remoting "tutorial"](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/running-remote-commands)
- [About Remote](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote)


[Get-Credential](https://docs.microsoft.com/en-us/powershell/module/Microsoft.PowerShell.Security/Get-Credential) enables you to get user credentials e.g. when __not__ using SSH for authentication:
```powershell
$cred = Get-Credential -UserName domain\username
```

This is most useful when remoting between a Windows client and host.  This can be passed to most commands below by adding `-Credential $cred`.

If using SSH and not using key-based authentication or needing a different username, most of the commands below also accept a `-UserName username` argument.

[Enter-PSSession](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enter-pssession) creates an interactive connection giving you a shell on a host.  This is comparable to ssh-ing into a machine:

```powershell
Enter-PSSession WindowsHostnameOrIP
# OR, if required by Linux/Mac client, explicitly specify to use SSH
Enter-PSSession WindowsHostnameOrIP -SSHTransport

# Leave the interactive session
Exit-PSSession
# OR
<ctrl+d>
```

Example:
```powershell
PS /Users/mac_user> Enter-PSSession win_hostname -SSHTransport
[win_hostname]: PS C:\Users\win_user\Documents>
```

[Invoke-Command](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/invoke-command) can be used to run commands or scripts on one or more hosts:

```powershell
# Execute command `Hostname` on host
Invoke-Command -ComputerName hostname -ScriptBlock { Hostname }
Invoke-Command -SSHTransport -HostName hostname -ScriptBlock { Hostname }

# Execute multiple commands on host
Invoke-Command -SSHTransport -HostName hostname -ScriptBlock { Hostname; Get-Date }

# Run remote script on host
Invoke-Command -ComputerName hostname -ScriptBlock { c:\path\remote.ps1 }
Invoke-Command -SSHTransport -HostName hostname -ScriptBlock { c:\path\remote.ps1 }

# Run local script on host
Invoke-Command -FilePath ./local.ps1 -ComputerName hostname
Invoke-Command -FilePath ./local.ps1 -SSHTransport -HostName hostname

# Execute command `Hostname` on multiple hosts
Invoke-Command -ComputerName hostname0, hostname1 -ScriptBlock { Hostname }
Invoke-Command -SSHTransport -HostName hostname0, hostname1 -ScriptBlock { Hostname }

# You can likewise run local/remote scripts on multiple hosts
```

Additional information about [ScriptBlock](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_script_blocks).

[New-PSSession](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/new-pssession) creates a "persistent" connection that can re-used to run multiple remote commands:

```powershell
# Create session to host
$session = New-PSSession -ComputerName hostname
$session = New-PSSession -SSHTransport -HostName hostname

# List sessions
Get-PSSession

# Re-use session to run multiple commands/scripts
Invoke-Command -Session $session -ScriptBlock { Hostname }
Invoke-Command -Session $session -ScriptBlock { Hostname; $d = Get-Date }
# Variables are created in each session and can subsequently be reused
Invoke-Command -Session $session -ScriptBlock { $d; c:\path\remote.ps1 }
Invoke-Command -Session $session -FilePath ./local.ps1

# Create session to multiple hosts
$session2 = New-PSSession -SSHTransport -HostName hostname1, hostname2
# Use multiple single-host sessions
Invoke-Command -Session $session,$session -ScriptBlock { Hostname }
# Use multiple multi-host sessions
Invoke-Command -Session ($session2+$session) -ScriptBlock { Hostname }
```

That last one is tricky.  `New-PSSession` with a single host returns a `PSSession`, but with multiple hosts returns an `Object[]` (array of `PSSession`s).  You'll have to concatenate them otherwise it fails with `Cannot convert 'System.Object[]' to the type 'System.Management.Automation.Runspaces.PSSession' required by parameter 'Session'`.

- [Get-PSSession](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-pssession)
- [Connect-PSSession](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/connect-pssession) ([unavailable on Mac/Linux](https://docs.microsoft.com/en-us/powershell/scripting/whats-new/known-issues-ps6#command-availability))

### Variables

Using [remote vs. local variables](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_variables) is surprisingly painless.  To use a local variable prefix the name with `$using:`, otherwise it's remote:
```powershell
Invoke-Command -Session $session -ScriptBlock { $home; $using:home }
# Output
C:\Users\win_user
/Users/mac_user
```

## Additional Resources

- [PowerShell Remoting From macOS To Windows Server](https://garrettyamada.com/powershell-remoting/)
- [PowerShell 7 RSA keys SSH Remoting](https://dev.to/v2kiran/powershell-7-rsa-keys-ssh-remoting-3m7e)