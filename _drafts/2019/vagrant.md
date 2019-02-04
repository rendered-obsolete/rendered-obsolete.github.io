---
layout: post
title: Migrating to Vagrant
tags:
- devops
- vagrant
---

Need testing various version of Windows: 7, 10, 10 LTSB ("IoT Enterprise").

Previously Jenkins [node/agent label](https://jenkins.io/doc/book/pipeline/syntax/#agent) physical machines that were manually configured:
- Scalability: static set of machines that became a bottleneck before releases
- Fail-fast/Debuggable: had to wait for PR to percolate through build pipeline to find out it didn't work and then take the node offline to debug
- Reproducable: manually configured error-prone, people would make changes to the environment, etc.

[AppVeyor]({% post_url /2018/2018-09-21-dotnet %})
[Travis]({% post_url /2018/2018-09-30-rust-ffi-ci %})


## Windows VM

Vagrant Windows guest VMs
https://www.vagrantup.com/docs/boxes/base.html#windows-boxes

We're in the process of evaluating Windows 10 Enterprise LTSC (the OS formerlly known as Windows 10 "IoT Enterprise"/LTSB) which is comparable to Windows 10 Enterprise version 1809/RS5.  I'm installing to VirtualBox as `win10_ltsc_2019`.

### WinRM

[Windows Remote Mangement](https://docs.microsoft.com/en-us/windows/desktop/WinRM/portal)

Make current network connection "Private" otherwise the WinRM configuration will fail: 
```powershell
Get-NetConnectionProfile
# Get "Name" value from output
Set-NetConnectionProfile -Name "Name from above" -NetworkCategory Private
```

`vagrant resume` there's a problem with WinRM:
```
default: WinRM transport: negotiate
```

In my case I'm on a domain and the connection will eventually rename itself.  Once that happens I need to set the category to "Private" again.

### RDP/SSH

[Enable Remote Desktop](https://docs.microsoft.com/en-us/windows-server/remote/remote-desktop-services/clients/remote-desktop-allow-access) for `vagrant rdp` to work.

In order for `vagrant ssh` to work, must install OpenSSH server:
- Windows 10 RS3/1709 and later, it's an [optional component](https://blogs.msdn.microsoft.com/powershell/2017/12/15/using-the-openssh-beta-in-windows-10-fall-creators-update-and-windows-server-1709/)
- For ealier versions including Windows 10 LTSB 2016 (RS1), https://github.com/PowerShell/Win32-OpenSSH/releases

In either case, you also need to:
- [Add the vagrant public key](https://www.vagrantup.com/docs/boxes/base.html#default-user-settings) to `~/.ssh/authorized_keys` (see [this SO](https://stackoverflow.com/questions/16212816/setting-up-openssh-for-windows-using-public-key-authentication))
- Have sshd start automatically: `sc config sshd start= auto`

### Boxing

At this point a PowerShell guru would probably head right into provisioning.  [With AppVeyor]({% post_url /2018/2018-09-21-dotnet %}#code-coverage) I got some mileage out of [Chocolatey](https://chocolatey.org/), but my shell-fu is woefully inadequate to fully automate Windows.

A very common Visual Studio:
- [Full Visual Studio](https://visualstudio.microsoft.com/downloads/)
- ["BuildTools"](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2017) (a streamlined version of VS)
- [Visual Studio container](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container?view=vs-2017)


Working with boxes is straight-forward.  In the case of virtual box:
https://www.vagrantup.com/docs/virtualbox/boxes.html#packaging-the-box
```bash
# Save the base image somewhere
vagrant package --base win10_ltsc_2019 --output /shared_folder/win10_ltsc_2019.box

# Remove the box if it already exists
vagrant box remove win10_ltsc_2019
# Add the saved base image to list of available "boxes"
vagrant box add --name win10_ltsc_2019 /shared_folder/win10_ltsc_2019.box
```

Keep in mind none of this is fast as it involves 5-15GB VM images.  This should be used for relatively "static" configuration; I wouldn't want to mess with it every day or even every week.  For things that change frequently (like your software) there's "shared folders".

Neat scripts and Vagrantfiles in this repo for working with Windows and Docker:
https://github.com/StefanScherer/docker-windows-box


## Vagrantfile

```ruby
Vagrant.configure("2") do |config|
    config.vm.box = "win10_ltsc_2019"
    config.vm.guest = :windows
    config.vm.communicator = "winrm"
    # 3389 RDP
    config.vm.network "forwarded_port", guest: 3389, host: 3389
end
```

`vagrant up`:
```
==> default: Forwarding ports...
    default: 3389 (guest) => 3389 (host) (adapter 1)
    default: 5985 (guest) => 55985 (host) (adapter 1)
    default: 5986 (guest) => 55986 (host) (adapter 1)
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Mounting shared folders...
    default: /vagrant => /Users/jake/projects/nng.NETCore
```

Port `3389` is RDP so `vagrant rdp` works.  I added a (very belated) [answer to Stack Overflow](https://stackoverflow.com/questions/28906432/vagrant-rdp-windows2012r2-how-do-i-rdp-into-my-vagrant-box) about this:

```
$ vagrant rdp
==> default: Detecting RDP info...
    default: Address: 127.0.0.1:3389
    default: Username: vagrant
```

Likewise, `vagrant ssh`:  
![]({{ "/assets/vagrant_ssh_win10.png" | absolute_url }})

