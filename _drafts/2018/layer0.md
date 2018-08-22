---
layout: post
title: More Windows Service in C#
tags:
- win10
- csharp
- service
- pinvoke
---

[Previously]({% post_url /2018/2018-08-21-windows-services %}) we created a Windows service we call "layer0".

Our application has the additional wrinkle that this service needs to interact with the user and their desktop.  [Interactive Services](https://docs.microsoft.com/en-us/windows/desktop/Services/interactive-services) provides guidance how to accomplish this.  Basically, spawn a desktop application as the user and use [IPC](https://en.wikipedia.org/wiki/Inter-process_communication) to communicate between the two.  We refer to this portion of our client as "layer1".

Owing to [XXXXXXXX our use of]() [Apache Thrift](https://thrift.apache.org/) we a complete messaging solution for IPC between layer1 and layer0 (the service).

SERVICE_INTERACTIVE_PROCESS

## Session Events

In order for layer0 service to spawn layer1 process as the user, we need to track user login/logout activity.  Modify [class derived from ServiceBase]({% post_url /2018/2018-08-21-windows-services %}#the-service).

First, set [ServiceBase.CanHandleSessionChangeEvent](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.canhandlesessionchangeevent):
```csharp
public Layer0Service()
{
    CanHandleSessionChangeEvent = true;
}
```

Then override [OnSessionChange](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onsessionchange):
```csharp
protected override void OnSessionChange(SessionChangeDescription changeDescription)
{
    Log("OnSessionChange: " + changeDescription.Reason);
    base.OnSessionChange(changeDescription);
    switch (changeDescription.Reason)
    {
        case SessionChangeReason.SessionLogon:
        case SessionChangeReason.SessionUnlock: // Switch between logged in users.
            DoLogon();
            break;
        case SessionChangeReason.SessionLogoff:
        //case SessionChangeReason.SessionLock:
            DoLogoff();
            break;
    }
}
```

The `Logon` and `Logoff` events are self-explanatory.  `Unlock` (and the corresponding `Lock`) less so.  Unlock events occur when fast user switching is used, for example.

## Start Process as Desktop User

PInvoke ahoy:
```csharp
#if DEBUG
const Pinvoke.CreationFlags Layer1CreationFlags = Pinvoke.CreationFlags.CreateNewConsole;
#else
const Pinvoke.CreationFlags Layer1CreationFlags = Pinvoke.CreationFlags.CreateNoWindow;
#endif

Pinvoke.PROCESS_INFORMATION? serviceStartLayer1AsUser(string exePath, string command)
{
    IntPtr server = IntPtr.Zero;
    IntPtr ppSessionInfo = IntPtr.Zero;
    IntPtr userToken = IntPtr.Zero; // WARNING: the user token is supposed to be a secret, don't print it anywhere
    try
    {
        // Query all sessions on local machine
        server = Pinvoke.WTSOpenServer("localhost");
        Int32 count = 0;
        Int32 retval = Pinvoke.WTSEnumerateSessions(server, ref ppSessionInfo, ref count);
        if (retval == 0)
        {
            throw new Win32Exception(Marshal.GetLastWin32Error(), "WTSEnumerateSessions");
        }

        // Find session for the logged in user and get their token
        Int32 dataSize = Marshal.SizeOf(typeof(Pinvoke.WTS_SESSION_INFO));
        Int64 current = (Int64)ppSessionInfo;
        for (int i = 0; i < count; i++)
        {
            var si = (Pinvoke.WTS_SESSION_INFO)Marshal.PtrToStructure((IntPtr)current, typeof(Pinvoke.WTS_SESSION_INFO));
            current += dataSize;

            if (si.State != Pinvoke.WTS_CONNECTSTATE_CLASS.WTSActive && si.State != Pinvoke.WTS_CONNECTSTATE_CLASS.WTSConnected)
                continue;

            var sessionId = (uint)si.SessionID;
            // WARNING: the user token is supposed to be a secret, don't print it anywhere
            if (OS.Pinvoke.WTSQueryUserToken(sessionId, out userToken))
            {
                Log(LogLevel.Info, "WTSQueryUserToken succeeded for session: " + sessionId);
                break;
            }
        }

        if (userToken == IntPtr.Zero)
        {
            Log(LogLevel.Error, "Unable to obtain user token, unable to start layer1");
            return null;
        }

        // Launch layer1 as the logged-in user
        var nullSecurityAttributes = new Pinvoke.SECURITY_ATTRIBUTES { lpSecurityDescriptor = IntPtr.Zero };
        Pinvoke.STARTUPINFO startupInfo = Pinvoke.StartupInfoAlloc();
        Pinvoke.PROCESS_INFORMATION processInfo;
        // WARNING: the user token is supposed to be a secret, don't print it anywhere
        Pinvoke.CreateProcessAsUser(userToken, exePath, command,
            ref nullSecurityAttributes, ref nullSecurityAttributes,
            true, Layer1CreationFlags, IntPtr.Zero, null, ref startupInfo, out processInfo);

        return processInfo;
    }
    finally
    {
        if (server != IntPtr.Zero)
            Pinvoke.WTSCloseServer(server);
        if (ppSessionInfo != IntPtr.Zero)
            Pinvoke.WTSFreeMemory(ppSessionInfo);
        if (userToken != IntPtr.Zero)
            Pinvoke.CloseHandle(userToken);
    }
}
```

Highlights:
1. Use [WTSEnumerateSessions](https://docs.microsoft.com/en-us/windows/desktop/api/wtsapi32/nf-wtsapi32-wtsenumeratesessionsa) to iterate over all sessions looking for the active desktop session.
1. [WTSQueryUserToken](https://docs.microsoft.com/en-us/windows/desktop/api/wtsapi32/nf-wtsapi32-wtsqueryusertoken) to obtain primary access token for that session's user.
1. [CreateProcessAsUser](https://msdn.microsoft.com/en-us/library/windows/desktop/ms682429%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396) to launch a process (i.e. layer1) as that user.

Now both the layer0 service and layer1 process are running.

## Console Executable

Quick aside.

When a developer is running layer0 as a desktop console application instead of a service, starting layer1 is more straightforward:
```csharp
Pinvoke.PROCESS_INFORMATION? startLayer1(string exePath, string commandline)
{
    var nullSecurityAttributes = new Pinvoke.SECURITY_ATTRIBUTES { lpSecurityDescriptor = IntPtr.Zero };
    Pinvoke.STARTUPINFO startupInfo = Pinvoke.StartupInfoAlloc();
    Pinvoke.PROCESS_INFORMATION procInfo;
    Pinvoke.CreateProcess(exePath, commandline,
        ref nullSecurityAttributes, ref nullSecurityAttributes,
        true, Layer1CreationFlags, IntPtr.Zero, null,
        ref startupInfo, out procInfo);
    return procInfo;
}
```

The canonical solution to start another exe is [`System.Diagnostics.Process`](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process).  However, by using [CreateProcess](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessa) (which also returns [PROCESS_INFORMATION](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/ns-processthreadsapi-_process_information)), it means managing layer1 will be the same whether starting it from a service or console application.

## Process Wrangling

Once the layer1 process is running we've got a lot of behaviour and handling that is specific to our application.  The most interesting bit is the monitoring that the layer0 service does of layer1:

```csharp
var objState = (ObjectState)Pinvoke.WaitForSingleObject(layer1ProcInfo.hProcess, 0);
if (user_logout)
{
    // There's no longer a user
    if (objState == Pinvoke.ObjectState.WaitObject0)
    {
        // Already not running.  Wait for a user to login.
    }
    else if (objState == Pinvoke.ObjectState.WaitTimeout)
    {
        // Running.  Tell it to stop.
    }
}
else if (user_login_or_fast_user_switching)
{
    // There's a user (may or may not have been one before)
    if(objState == Pinvoke.ObjectState.WaitObject0)
    {
        // Not running, so start it.
    }
    else if(objState == Pinvoke.ObjectState.WaitTimeout)
    {
        // Running as different user.  Stop it, then restart it as new user
    }
}
else
{
    // Nothing special has happened.  This is the state we're normally in.
    if (objState == Pinvoke.ObjectState.WaitObject0)
    {
        // Process not running.  Check the exit code to see if it was intentional.
        if (Pinvoke.GetExitCodeProcess(procInfo.hProcess, out uint lpExitCode) && lpExitCode == (uint)L1ExitCode.UserLoggingOut)
        {
            // Process exited but it was told to because user is logging out.  We're probably going to receive a "user logout" event.
        }
        else
        {
            // Process stopped/crashed.  Restart it.
        }
    }
    else if (objState == Pinvoke.ObjectState.WaitTimeout)
    {
        // Still running.  Everything ok- do nothing.
    }
}
```

Here we use [`WaitForSingleObject()`](https://docs.microsoft.com/en-us/windows/desktop/api/synchapi/nf-synchapi-waitforsingleobject) to monitor layer1 process.  The first argument is the layer1 process handle, the second argument is `0` so the function returns immediately with the current state of the process:
- `WaitTimeout` means it's running
- `WaitObject0` means it's not

`GetExitCodeProcess()` is a fairly recent addition.  Looking at the log output we noticed that when a user is logging out layer1 process was exiting and the service tries unsuccessfully to restart it; it either fails to start or immediately exits.  This is similar to shell.  Details will follow when we talk about layer1.