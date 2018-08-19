---
layout: post
title: Windows Service in C#
tags:
- win10
- csharp
- service
- pinvoke
---

Part of our client runs as a Windows service for a few reasons:
- It can automatically start when Windows 10 boots
- OS can restart it if it fails
- It will be running even when no user is logged in
- When required, we have elevated privileges

Other than operations requiring elevated privileges, all those reasons only exist in a production environment.  During development we want the convenience of launching/debugging from Visual Studio and easily viewing stdout/stderr, so we also want it to function as a console application.

In our codebase and documentation this program is referred to as "layer0".

This and other Windows programming makes heavy use of [PInvoke](https://docs.microsoft.com/en-us/cpp/dotnet/how-to-call-native-dlls-from-managed-code-using-pinvoke).  [http://pinvoke.net/](http://pinvoke.net/) is indispensible.

## The Service

The key to creating a Windows service in C# is inheriting from 
[System.ServiceProcess.ServiceBase](https://msdn.microsoft.com/en-us/library/system.serviceprocess.servicebase(v=vs.110).aspx).

```csharp
class Layer0Service : System.ServiceProcess.ServiceBase
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

`DoStartup()` and `DoShutdown()` are routines that take care of startup and shutdown and are shared by the service and console application.

## The Main()

Our program entry point:
```csharp
static public int Main(string[] args)
{
    // Parse args

    if (install)
    {
        return Layer0Service.InstallService();
    }
    else if (service)
    {
        // Start as Windows service when run with --service
        var service = new Layer0Service();
        System.ServiceProcess.ServiceBase.Run(service);
        return 0;
    }

    // Running as a console program
    DoStartup();

    ...
```

This executable has 3 modes:
- `console.exe --install` installs the service
- `console.exe --service` executes as the service
- `console.exe` executes as a normal console application

## Service Installation

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
            // Start console exe with --service arg
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

[CreateService()](https://docs.microsoft.com/en-us/windows/desktop/api/winsvc/nf-winsvc-createservicea) is the main call.  `SERVICE_AUTO_START` means the service will start automatically.  The application registers itself and it will be passed the `--service` command line argument.

This places it in services.msc where __Run__ executes `console.exe --service`:

## setPermissions()

Our platform being a non-critical to the system, we felt that ordinary users should be able to start/stop it.

Reference these Stack Overflow issues:
- https://stackoverflow.com/questions/15771998/how-to-give-a-user-permission-to-start-and-stop-a-particular-service-using-c-sha
- https://stackoverflow.com/questions/8379697/start-windows-service-from-application-without-admin-rightc

```csharp
var serviceControl = new ServiceController(ShortServiceName);
var psd = new byte[0];
uint bufSizeNeeded;
bool ok = QueryServiceObjectSecurity(serviceControl.ServiceHandle, System.Security.AccessControl.SecurityInfos.DiscretionaryAcl, psd, 0, out bufSizeNeeded);
if (!ok)
{
    int err = Marshal.GetLastWin32Error();
    if (err == 0 || err == (int)ErrorCode.ERROR_INSUFFICIENT_BUFFER)
    {
        // Resize buffer and try again
        psd = new byte[bufSizeNeeded];
        ok = QueryServiceObjectSecurity(serviceControl.ServiceHandle, System.Security.AccessControl.SecurityInfos.DiscretionaryAcl, psd, bufSizeNeeded, out bufSizeNeeded);
    }
}
if (!ok)
{
    Log("Failed to GET service permissions", Logging.LogLevel.Warn);
}

// Give permission to control service to "interactive user" (anyone logged-in to desktop)
var rsd = new RawSecurityDescriptor(psd, 0);
var dacl = new DiscretionaryAcl(false, false, rsd.DiscretionaryAcl);
//var sid = new SecurityIdentifier("D:(A;;RP;;;IU)");
var sid = new SecurityIdentifier(WellKnownSidType.InteractiveSid, null);
dacl.AddAccess(AccessControlType.Allow, sid, (int)SERVICE_ACCESS.SERVICE_ALL_ACCESS, InheritanceFlags.None, PropagationFlags.None);

// Convert discretionary ACL to raw form
var rawDacl = new byte[dacl.BinaryLength];
dacl.GetBinaryForm(rawDacl, 0);
rsd.DiscretionaryAcl = new RawAcl(rawDacl, 0);
var rawSd = new byte[rsd.BinaryLength];
rsd.GetBinaryForm(rawSd, 0);

// Set raw security descriptor on service
ok = SetServiceObjectSecurity(serviceControl.ServiceHandle, SecurityInfos.DiscretionaryAcl, rawSd);
if (!ok)
{
    Log("Failed to SET service permissions", Logging.LogLevel.Warn);
}
```

## Failure Actions

This is a particularly heinous bit of pinvoke.  Blame falls squarely on the function we need [ChangeServiceConfig2()](https://docs.microsoft.com/en-us/windows/desktop/api/winsvc/nf-winsvc-changeserviceconfig2a) because its second parameter specifies what type the third parameter is a pointer to an array of.

We heavily consulted and munged together the following sources:
- [pinvoke.net entry for ChangeServiceConfig2()](http://pinvoke.net/default.aspx/advapi32/ChangeServiceConfig2.html)
- [This old MSDN blog](https://blogs.msdn.microsoft.com/anlynes/2006/07/30/using-net-code-to-set-a-windows-service-to-automatically-restart-on-failure/)
- [This MSDN code](https://code.msdn.microsoft.com/windowsdesktop/CSWindowsServiceRecoveryPro-2147e7ac), specifically:
    - [ServiceRecoveryProperty.cs](https://code.msdn.microsoft.com/windowsdesktop/CSWindowsServiceRecoveryPro-2147e7ac/sourcecode?fileId=21765&pathId=1975102639)
    - [Win32.cs](https://code.msdn.microsoft.com/windowsdesktop/CSWindowsServiceRecoveryPro-2147e7ac/sourcecode?fileId=21765&pathId=1037126777)


```csharp
public static void SetServiceRecoveryActions(IntPtr hService, params SC_ACTION[] actions)
{
    bool needsShutdownPrivileges = actions.Any(action => action.Type == SC_ACTION_TYPE.RebootComputer);
    if (needsShutdownPrivileges)
    {
        GrantShutdownPrivilege();
    }

    var sizeofSC_ACTION = Marshal.SizeOf(typeof(SC_ACTION));
    IntPtr lpsaActions = IntPtr.Zero;
    IntPtr lpInfo = IntPtr.Zero;
    try
    {
        // Setup array of actions
        lpsaActions = Marshal.AllocHGlobal(sizeofSC_ACTION * actions.Length);
        var ptr = lpsaActions.ToInt64();
        foreach (var action in actions)
        {
            Marshal.StructureToPtr(action, (IntPtr)ptr, false);
            ptr += sizeofSC_ACTION;
        }

        // Configuration parameters
        var serviceFailureActions = new SERVICE_FAILURE_ACTIONS
        {
            dwResetPeriod = (int)TimeSpan.FromDays(1).TotalSeconds,
            lpRebootMsg = null,
            lpCommand = null,
            cActions = actions.Length,
            lpsaActions = lpsaActions,
        };
        lpInfo = Marshal.AllocHGlobal(Marshal.SizeOf(serviceFailureActions));
        Marshal.StructureToPtr(serviceFailureActions, lpInfo, false);

        if (!ChangeServiceConfig2(hService, InfoLevel.SERVICE_CONFIG_FAILURE_ACTIONS, lpInfo))
        {
            throw new Win32Exception(Marshal.GetLastWin32Error());
        }
    }
    finally
    {
        if (lpsaActions != IntPtr.Zero)
            Marshal.FreeHGlobal(lpsaActions);
        if (lpInfo != IntPtr.Zero)
            Marshal.FreeHGlobal(lpInfo);
    }
}
```

The majority of this is setting up a [`SERVICE_FAILURE_ACTIONS`](https://docs.microsoft.com/en-us/windows/desktop/api/winsvc/ns-winsvc-_service_failure_actionsa) for the call to `ChangeServiceConfig2()`.  As mentioned in its documentation, if the service controller handles `SC_ACTION_TYPE.RebootComputer` the caller must have `SE_SHUTDOWN_NAME` privilege.  This is fulfilled by `GrantShutdownPrivilege()`.

Usage is reasonable:
```csharp
SetServiceRecoveryActions(hService,
    new SC_ACTION { Type = SC_ACTION_TYPE.RestartService, Delay = oneMinuteInMs },
    new SC_ACTION { Type = SC_ACTION_TYPE.RebootComputer, Delay = oneMinuteInMs },
    new SC_ACTION { Type = SC_ACTION_TYPE.None, Delay = 0 }
    );
```

`GrantShutdownPrivilege()` is pretty much taken verbatim from [MSDN code](https://code.msdn.microsoft.com/windowsdesktop/CSWindowsServiceRecoveryPro-2147e7ac/sourcecode?fileId=21765&pathId=1975102639):
```csharp
static void GrantShutdownPrivilege()
{
    IntPtr hToken = IntPtr.Zero;
    try
    {
        // Open the access token associated with the current process.
        var desiredAccess = System.Security.Principal.TokenAccessLevels.AdjustPrivileges | System.Security.Principal.TokenAccessLevels.Query;
        if (!OpenProcessToken(System.Diagnostics.Process.GetCurrentProcess().Handle, (uint)desiredAccess, out hToken))
        {
            throw new Win32Exception(Marshal.GetLastWin32Error());
        }
        // Retrieve the locally unique identifier (LUID) for the specified privilege.
        var luid = new LUID();
        if (!LookupPrivilegeValue(null, SE_SHUTDOWN_NAME, ref luid))
        {
            throw new Win32Exception(Marshal.GetLastWin32Error());
        }

        TOKEN_PRIVILEGE tokenPrivilege;
        tokenPrivilege.PrivilegeCount = 1;
        tokenPrivilege.Privileges.Luid = luid;
        tokenPrivilege.Privileges.Attributes = SE_PRIVILEGE_ENABLED;

        // Enable privilege in specified access token.
        if (!AdjustTokenPrivilege(hToken, false, ref tokenPrivilege))
        {
            throw new Win32Exception(Marshal.GetLastWin32Error());
        }
    }
    finally
    {
        if (hToken != IntPtr.Zero)
            CloseHandle(hToken);
    }
}
```

Alternatively, this can be done with [sc.exe](https://support.microsoft.com/en-us/help/251192/how-to-create-a-windows-service-by-using-sc-exe):
```
sc.exe failure Layer0 actions= restart/60000/restart/60000/""/60000 reset= 86400
```

I leave it as an exercise to the reader to guess which we're using more.