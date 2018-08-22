---
layout: post
title: Layer1
tags:
- win10
- csharp
- service
- pinvoke
---

I've covered "layer0", a Windows service that among other things spawns and monitors a process as the logged in user.  It only makes sense to talk about "layer1"

## Session Events

```csharp
static L1ExitCode ExitCode = L1ExitCode.RestartLayer1;

static int Main(string[] args)
{
    try
    {
        // These events only work if we have a Windows message pump
        Microsoft.Win32.SystemEvents.SessionEnding += SessionEnding;
        Microsoft.Win32.SystemEvents.SessionEnded += SessionEnded;
        Microsoft.Win32.SystemEvents.SessionSwitch += SessionSwitch;

        RunLayer1();
    }
    finally
    {
        Microsoft.Win32.SystemEvents.SessionEnding -= SessionEnding;
        Microsoft.Win32.SystemEvents.SessionEnded -= SessionEnded;
        Microsoft.Win32.SystemEvents.SessionSwitch -= SessionSwitch;
    }
    return (int)ExitCode;
}

static void SessionEnding(object sender, Microsoft.Win32.SessionEndingEventArgs args) => sessionEvent(SessionEvent.SessionEnding);
static void SessionEnded(object sender, Microsoft.Win32.SessionEndedEventArgs args) => sessionEvent(SessionEvent.SessionEnded);
static void SessionSwitch(object sender, Microsoft.Win32.SessionSwitchEventArgs args) => sessionEvent(SessionEvent.SessionSwitch);

static void sessionEvent(SessionEvent sessionEvent)
{
    switch(sessionEvent)
    {
        case SessionEvent.SessionEnded:
        case SessionEvent.SessionEnding:
            ExitCode = L1ExitCode.UserLoggingOut;
            // Trigger layer1 shutdown
            break;
        case SessionEvent.SessionSwitch:
        default:
            // Do nothing
            break;
    }
}
```

When a user logs out, first `SessionEvent.SessionEnding` followed shortly by `SessionEvent.SessionEnded`.

The [docs for `Microsoft.Win32.SystemEvents`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.win32.systemevents) state the events we're interested in are only raised if the message pump is running.  It just so happens that layer1 has to have a message loop.

## Message Loop

One of the oddities of our platform is that the [EC](https://en.wikipedia.org/wiki/Embedded_controller) driver delivers power button presses as low-level keyboard events.

Install hook and start message loop:
```csharp
private delegate IntPtr LowLevelKeyboardProc(int nCode, IntPtr wp, IntPtr lp);
LowLevelKeyboardProc lpfn;
IntPtr hHook;

bool setHWInterface2()
{
    cts = new CancellationTokenSource();
    keyboardHookProcTask = Task.Factory.StartNew((_) =>
    {
        lpfn = new LowLevelKeyboardProc(KeyboardHookProc);

        var moduleHandle = GetModuleHandle(null);
        //using (var curModule = System.Diagnostics.Process.GetCurrentProcess().MainModule)
        //{
        //    var moduleHandle = GetModuleHandle(curModule.ModuleName);
        //    // SetWindowsHookEx()
        //}
        hHook = SetWindowsHookEx(WH_KEYBOARD_LL, lpfn, moduleHandle, 0);
        if (hHook == IntPtr.Zero)
        {
            Log("SetWindowsHookEx failed", LogLevel.Error);
        }
        else
        {
            // SetWindowsHookEx requires a Windows message loop on the thread

            winMsgLoopAppCtx = new ApplicationContext();
            Application.Run(winMsgLoopAppCtx);
            // Using Application.DoEvents() seems cleaner, but hook function wasn't getting called
            //while (!cts.IsCancellationRequested)
            //{
            //    Application.DoEvents();
            //    // Sleep long enough this thread rarely runs, but short enough the button will be responsive
            //    await Task.Delay(250, cts.Token);
            //}

            if (!UnhookWindowsHookEx(hHook))
                Log("UnhookWindowsHookEx failed", LogLevel.Warn);
        }
    }, cts.Token, TaskCreationOptions.LongRunning);
    return true;
}
```

The hook callback passed to [SetWindowsHookEx](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644990%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396) also see [LowLevelKeyboardProc](https://msdn.microsoft.com/en-us/library/windows/desktop/ms644990%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396).

[ApplicationContext](https://docs.microsoft.com/en-us/dotnet/api/system.windows.forms.applicationcontext) to [Application.Run()](https://docs.microsoft.com/en-us/dotnet/api/system.windows.forms.application.run#System_Windows_Forms_Application_Run_System_Windows_Forms_ApplicationContext_)

To later stop it:
```csharp
winMsgLoopAppCtx.ExitThread();
```

Hook callback based off [pinvoke.net sample](https://www.pinvoke.net/default.aspx/Structures/KBDLLHOOKSTRUCT.html):
```csharp
IntPtr KeyboardHookProc(int code, IntPtr wParam, IntPtr lParam)
{
    if (code >= 0)
    {
        var kbScan = (KBDLLHOOKSTRUCT)Marshal.PtrToStructure(lParam, typeof(KBDLLHOOKSTRUCT));

        // Filter spurious power button events, play audio feedback via PC speaker, etc.
    }

    // Ensure other applications that have installed hooks receive hook notifications
    return CallNextHookEx(hHook, code, wParam, lParam);
}
```

## Big Red Button

During development both layer0 and layer1 are usually running as console applications.  Because [XXXXXXXX layer0 will restart layer1](), we originally had to stop layer0 then stop layer1.  This was pretty annoying so we changed it so both close if either windows is closed:

```csharp
static OS.Pinvoke.ConsoleCtrlDelegate ConsoleCtrlDelegate;

public static void StartLayerN(ILayerN layer)
{
    // If user hits ctrl-c or closes console window, shut everything down
    ConsoleCtrlDelegate += ctrlTypes =>
    {
        switch (ctrlTypes)
        {
            case OS.Pinvoke.CtrlTypes.CTRL_C_EVENT:
            case OS.Pinvoke.CtrlTypes.CTRL_CLOSE_EVENT:
                // Tell layer0 and layer1 to exit
                
                // Need to wait here otherwise OS terminates the process (without waiting for layer1, etc.)
                layer.Wait();
                // We handled the signal and thus return true as per https://docs.microsoft.com/en-us/windows/console/handlerroutine
                return true;
        }
        return false;
    };
    OS.Pinvoke.SetConsoleCtrlHandler(ConsoleCtrlDelegate, true);

    //...
}
```