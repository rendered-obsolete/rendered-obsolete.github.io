---
layout: post
title: Subtleties of async/await in C#
tags:
- csharp
- async
- performance
---

## Jargon

Parallelism- 
Concurrency- 

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

- Task provides [`Result` property](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task-1.result?view=netframework-4.7.2) and [`Wait()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.wait?view=netframework-4.7.2), but [you might not want to use them](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html).

- You can start tasks with [`TaskFactory.StartNew()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskfactory.startnew?view=netframework-4.7.2), but [you might not want to](https://blogs.msdn.microsoft.com/pfxteam/2011/10/24/task-run-vs-task-factory-startnew/) ([also see](https://blog.stephencleary.com/2013/08/startnew-is-dangerous.html)).

- [Functions should return](https://blogs.msdn.microsoft.com/pfxteam/2012/02/08/potential-pitfalls-to-avoid-when-passing-around-async-lambdas/) `async Task` __NOT__ `async void`.  [Unless it's an event](https://channel9.msdn.com/Series/Three-Essential-Tips-for-Async/Tip-1-Async-void-is-for-top-level-event-handlers-only).

- `StartNew(..., cancellationTokenSource.Token);` is [a task that might not start, not a cancellable task](https://blog.stephencleary.com/2015/03/a-tour-of-task-part-9-delegate-tasks.html).

- Prefer ["slim" variants](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim?view=netframework-4.7.2) over ["fat" ("non-slim"?) versions](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphore?view=netframework-4.7.2) of synchronization primitives



## It's Hard

The truth is writing this kind of code _correctly_ is non-trivial.  It's deep in the territory of "easy to learn but difficult to master".

Check out Stephen Toub's excellent series on "Building Async Coordination Primitives": [part1](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-1-asyncmanualresetevent/), [part2](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-2-asyncautoresetevent/), [part3](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-3-asynccountdownevent/), [part4](https://blogs.msdn.microsoft.com/pfxteam/2012/02/11/building-async-coordination-primitives-part-4-asyncbarrier/), [part5](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-5-asyncsemaphore/), [part6](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-6-asynclock/), [part7](https://blogs.msdn.microsoft.com/pfxteam/2012/02/12/building-async-coordination-primitives-part-7-asyncreaderwriterlock/)


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

## Putting Things in Context

SyncContexts is where the mental model equating tasks to threads really starts to unravel.

UI frameworks are notoriously picky about where you do things.

https://docs.microsoft.com/en-us/dotnet/framework/winforms/controls/how-to-make-thread-safe-calls-to-windows-forms-controls

## Take-aways

__Read.  Read.  Read.__

There's tons of documents and resources.  The insight and experience of others is invaluable.  But make sure to get your hands dirty, too.

__Ride on the Wheels of Others__

__Check the source__

Not the code, the author.  The internet is full of well-intentioned advice that "seems" to work, but is sometimes subtlely wrong.  If someone isn't an authoritative expert (either the author of a book on the subject, a member of high-profile open-source/Microsoft project, etc.)- reader beware.

Actually, checking the source code is good advice too.

In short, async/await is a powerful tool in the C# toolbox, but not a silver bullet.  [When faced with a problem, you might think "I know, I'll use async/await".  Now you have two problems](https://blog.codinghorror.com/regular-expressions-now-you-have-two-problems/).