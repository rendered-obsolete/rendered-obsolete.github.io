---
layout: post
title: Windows Service in C#
tags:
- win10
- csharp
- service
---

We a portion of our client to run as a Windows service for a few reasons:
- It will automatically start when Windows 10 boots
- It will be restarted if it fails
- It will be running even when no user is logged in
- We can do things requiring elevated privileges

The key to creating a Windows service in C# is inheriting from 
[System.ServiceProcess.ServiceBase](https://msdn.microsoft.com/en-us/library/system.serviceprocess.servicebase(v=vs.110).aspx).

```csharp
class WinService : System.ServiceProcess.ServiceBase
{
    protected override void OnStart(string[] args)
    {
        base.OnStart(args);
        DoStartup();
    }

    protected override void OnStop()
    {
        base.OnStop();
        DoShutdown();
    }
}
```

Where `DoStartup()` and `DoShutdown()` are 

The `Main()` for our executable looks like:
```csharp
public class MainClass
{
    static public int Main(string[] args)
    {
        // Parse args

        if (install)
        {
            return WinService.InstallService();
        }
        else if (start)
        {
            
        }
        else if (service)
        {
            // Start as Windows service when run with --service
            var service = new WinService();
            System.ServiceProcess.ServiceBase.Run(service);
            return 0;
        }

        // Running as a console program
        DoStartup();
```

Installing our exe as a Windows service involves several pinvokes:
- OpenSCManager()
- CreateService()
- CloseServiceHandle()
```csharp
public static int InstallService()
{
    IntPtr hSC = IntPtr.Zero;
    IntPtr hService = IntPtr.Zero;
    try
    {
        hSC = OpenSCManager(null, null, SCM_ACCESS.SC_MANAGER_ALL_ACCESS);

        string fullPathFilename = null;
        using (var currentProc = Process.GetCurrentProcess())
        {
            fullPathFilename = currentProc.MainModule.FileName;
        }

        hService = CreateService(hSC, ShortServiceName, DisplayName,
            SERVICE_ACCESS.SERVICE_ALL_ACCESS, SERVICE_TYPE.SERVICE_WIN32_OWN_PROCESS, SERVICE_START.SERVICE_AUTO_START, SERVICE_ERROR.SERVICE_ERROR_NORMAL,
            // Start our exe with --service arg
            fullPathFilename + " --service",
            null, null, null, null, null
            );

        setPermissions();

        return 0;
    }
    catch (Exception ex)
    {
        // Error handling
        return -1;
    }
    finally
    {
        if (hService != IntPtr.Zero)
            CloseServiceHandle(hService);
        if (hSC != IntPtr.Zero)
            CloseServiceHandle(hSC);
    }
}
```