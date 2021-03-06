---
layout: post
title: Subtleties of async/await in C#
tags:
- csharp
- async
- performance
---


## It's a Thread

Except it isn't.

Read ["There is no Thread"](https://blog.stephencleary.com/2013/11/there-is-no-thread.html).  Seriously, read it.

The waters are muddied by the language/API:
```csharp
Task.Factory.StartNew(async () =>
{
    Thread.CurrentThread.Name = "XXXX";
    // Do work
}, TaskCreationOptions.LongRunning);
```

Here we used/abused [TaskCreationOptions.LongRunning](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcreationoptions?view=netframework-4.7.2) to request (and incidentally receive) a thread.

Although not quite as usable as the threads window, the tasks window (when debugging, __Debug->Windows->Tasks__) is helpful:

![]({{ "/assets/vs_tasks.png" | absolute_url }})

## Misled and Led-Astray

The continued existance of legacy APIs seemingly encourage incorrect behaviour.

From ["Async/Await - Best Practices in Asynchronous Programming"](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx?f=255&MSPPError=-2147217396) and other sources:

- Task provides [`Result` property](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task-1.result?view=netframework-4.7.2) and [`Wait()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.wait?view=netframework-4.7.2), but [you might not want to use them](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html).  Really, [don't use them](http://blog.stephencleary.com/2012/12/dont-block-in-asynchronous-code.html).

- You can start tasks with [`TaskFactory.StartNew()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskfactory.startnew?view=netframework-4.7.2), but [you might not want to](https://blogs.msdn.microsoft.com/pfxteam/2011/10/24/task-run-vs-task-factory-startnew/) ([also see](https://blog.stephencleary.com/2013/08/startnew-is-dangerous.html)).

- [Functions should return](https://blogs.msdn.microsoft.com/pfxteam/2012/02/08/potential-pitfalls-to-avoid-when-passing-around-async-lambdas/) `async Task` __NOT__ `async void`.  [Unless it's an event](https://channel9.msdn.com/Series/Three-Essential-Tips-for-Async/Tip-1-Async-void-is-for-top-level-event-handlers-only).

- `StartNew(..., cancellationTokenSource.Token);` is [a task that might not start, not a cancellable task](https://blog.stephencleary.com/2015/03/a-tour-of-task-part-9-delegate-tasks.html).

- Prefer ["slim" variants](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim?view=netframework-4.7.2) over ["fat" ("non-slim"?) versions](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphore?view=netframework-4.7.2) of synchronization primitives

Control.BeginInvoke method in Windows Forms, or the Dispatcher.BeginInvoke method in WPF
https://blogs.msdn.microsoft.com/pfxteam/2012/06/15/executioncontext-vs-synchronizationcontext/

## It's Hard

The truth is writing this kind of code _correctly_ is non-trivial.  It's deep in the territory of "easy to learn but difficult to master".

Check out Stephen Toub's excellent series on "Building Async Coordination Primitives": [part1](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-1-asyncmanualresetevent/), [part2](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-2-asyncautoresetevent/), [part3](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-3-asynccountdownevent/), [part4](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-4-asyncbarrier/), [part5](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-5-asyncsemaphore/), [part6](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-6-asynclock/), [part7](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-7-asyncreaderwriterlock/)

Ran into this:
https://stackoverflow.com/questions/19481964/calling-taskcompletionsource-setresult-in-a-non-blocking-manner
This simplified example deadlocks:
```csharp
[Theory]
[ClassData(typeof(TransportsClassData))]
public async Task ReqRepBasic(string url)
{
    using (var reqAioCtx = factory.RequesterCreate(url).CreateAsyncContext(factory)
    {
        var asyncReq = await reqAioCtx.Send(factory.CreateMessage());
    }
}
```

Callstack:
```
nng.NETCore.dll!nng.AsyncCtx<nng.IMessage>.Dispose(bool disposing) Line 170 (/Users/jake/projects/nng.NETCore/nng.NETCore/AsyncContext.cs:170)
nng.NETCore.dll!nng.AsyncBase<nng.IMessage>.Dispose() Line 68 (/Users/jake/projects/nng.NETCore/nng.NETCore/AsyncContext.cs:68)
tests.dll!nng.Tests.ReqRepTests.ReqRepBasic(string url) Line 35 (/Users/jake/projects/nng.NETCore/tests/ReqRepTests.cs:35)
[Resuming Async Method] (Unknown Source:0)
[External Code] (Unknown Source:0)
nng.NETCore.dll!nng.ReqAsyncCtx<nng.IMessage>.Send(nng.IMessage message) Line 42 (/Users/jake/projects/nng.NETCore/nng.NETCore/AsyncReqRep.cs:42)
[Resuming Async Method] (Unknown Source:0)
[External Code] (Unknown Source:0)
nng.NETCore.dll!nng.ReqAsyncCtx<nng.IMessage>.callback(System.IntPtr arg) Line 77 (/Users/jake/projects/nng.NETCore/nng.NETCore/AsyncReqRep.cs:77)
[External Code] (Unknown Source:0)
```

1. `await Send()`
1. Eventually, `callback()` (registered with [`nng_aio_alloc()`](https://nanomsg.github.io/nng/man/v1.0.0/nng_aio_alloc.3.html) via `CreateAsyncContext()`) is called from native code
1. In `callback()`, [`TaskCompletionSource.SetResult()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource-1.setresult?view=netframework-4.7.2) synchronously executes continuation of `await Send()`
1. Reach end of `using` block and call `Dispose()`
1. `AsyncCtx<nng.IMessage>.Dispose()` calls `nng_aio_free()` (native code that waits for async operations to complete and releases resources allocated with `nng_aio_alloc()`)

We're in `callback()` (#2) and because of the synchronous continuation (#3) execute `nng_aio_free()` (#5) on the same thread, which waits for #2...  Oops.

I ended up using `new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously)` to break #3.

## Cancellation

Stopping something is conceptually very simple.  Yet whether it's handling interrupts in the Linux kernel, or writing exception safe c++, in software it's consistently one of the most difficult things to get right.  Or at best riddled with caveats.

- [Task cancellation basics](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation):
    1. Create [`CancellationTokenSource`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource?view=netframework-4.7.2)
    1. `Token` property to pass [`CancellationToken`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken?view=netframework-4.7.2) to tasks
    1. `CancellationTokenSource.Cancel()`
- [`TaskFactory.StartNew()` variants accepting `CancellationToken`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskfactory.startnew?view=netframework-4.7.2) don't create a cancellable task (just the scheduling may be cancelled)
- [Intermediate details](https://docs.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads)
- [Canceling anything](https://blogs.msdn.microsoft.com/pfxteam/2012/10/05/how-do-i-cancel-non-cancelable-async-operations/)
- "Building Async Coordination Primitives" series made heavy use of [`TaskCompletionSource`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource-1?view=netframework-4.7.2) that [you might need to cancel](https://github.com/StephenClearyArchive/AsyncEx.Tasks/blob/master/src/Nito.AsyncEx.Tasks/CancellationTokenTaskSource.cs)

It's tempting to write:
```csharp
Task.Run(async () =>
{
    while( !cancellationToken.IsCancellationRequested )
    {
        // Do something...
    }
});
```

[Using `IsCancellationRequested` like this is logically correct (the task will end), but is contractually wrong](https://docs.microsoft.com/en-us/dotnet/standard/threading/how-to-listen-for-cancellation-requests-by-polling) because no [`OperationCanceledException`](https://docs.microsoft.com/en-us/dotnet/api/system.operationcanceledexception?view=netframework-4.7.2) is thrown.

For example, [`Task.ContinueWith(..., TaskContinuationOptions.OnlyOnCanceled)`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.continuewith?view=netframework-4.7.2) won't work.

Take this dull program (using `<LangVersion>7.2</LangVersion>` for async main):
```csharp
static async Task Main(string[] args)
{
    var cts = new CancellationTokenSource();
    cts.CancelAfter(100);
    await Task.Run(async () => {
        await Task.Delay(1000, cts.Token);
    });
}
```

`Task.Delay()` throws `System.Threading.Tasks.TaskCanceledException`.  One of the hard-truths I struggled with was the borderline cavalier use of exceptions for "ordinary" flow-control.


## Putting Things in Context

SyncContexts is where the mental model equating tasks to threads really starts to unravel.

UI frameworks are notoriously picky about where you do things.

https://docs.microsoft.com/en-us/dotnet/framework/winforms/controls/how-to-make-thread-safe-calls-to-windows-forms-controls

https://msdn.microsoft.com/en-us/magazine/gg598924.aspx?f=255&MSPPError=-2147217396

## Take-aways

__Read.  Read.  Read.__

There's tons of documents and resources.  The insight and experience of others is invaluable.  But make sure to get your hands dirty, too.

__Ride on the Wheels of Others__

__Check the source__

Not the code, the author.  The internet is full of well-intentioned advice that "seems" to work, but is sometimes subtlely wrong.  If someone isn't an authoritative expert (either the author of a book on the subject, a member of high-profile open-source/Microsoft project, etc.)- reader beware.

Actually, checking the source code is good advice too.

In short, async/await is a powerful tool in the C# toolbox, but not a silver bullet.  [When faced with a problem, you might think "I know, I'll use async/await".  Now you have two problems](https://blog.codinghorror.com/regular-expressions-now-you-have-two-problems/).