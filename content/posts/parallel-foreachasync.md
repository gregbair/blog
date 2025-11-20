---
title: Parallel.ForEachAsync Deep Dive
publishDate: 2021-12-12T00:00:00+00:00
---
> This post is part of the [2021 C# Advent calendar](https://csadvent.christmas/). Check it out for more C# goodness!

## Intro

> This post contains a very technical dive. It is of intermediate complexity, and assumes a basic knowledge of how async/await works.

.NET 6 introduced a small feature that's important, but was probably overlooked - [`Parallel.ForEachAsync`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync?view=net-6.0).

This is a (IMO) pretty powerful and elegant implementation of something that's been missing for a while from .NET - parallel processing of async operations over a collection.

`Parallel.ForEach` has been in the BCL for a long time, but there hasn't been a good way to do something similar for async operations on collections.

## Usage

The basic usage of this method is thus:

```csharp
var nums = Enumerable.Range(0, 100).ToArray();

await Parallel.ForEachAsync(nums, async (i, token) =>
{
    Console.WriteLine($"Starting iteration {i}");
    await Task.Delay(1000, token);
    Console.WriteLine($"Finishing iteration {i}");
});
```

What this gives you is a basic foreach loop over your collection, in parallel, with safeguards in place to not overwhelm your system (kinda).

> NOTE: I would not actually use the `ForEachAsync<TSource>(IAsyncEnumerable<TSource>, Func<TSource,CancellationToken,ValueTask>)` overload of this method. Below is why the better, safer option is `ForEachAsync<TSource>(IEnumerable<TSource>, ParallelOptions, Func<TSource,CancellationToken,ValueTask>)`.

In the invocation above, we're not specifying a _degree of parallelism (DoP)_ constraint, meaning we're not saying how many iterations of the lambda to run at once. So what happens? You get the default, which is the number returned by `Environment.ProcessorCount`. That property, though, is misnamed, as it's not the number of processors, but the number of _cores_ your system has. (Mine shows 16, as I have an AMD Ryzen 7, which has 8 physical cores + hyperthreading).

The safer thing to do is to specify a max DoP. The right number is the classic developer answer "it depends". It depends on your system, the size of your collection, a host of factors that are outside the scope of this post. Your best bet is to make it configurable and test a lot of different values.

You specify that using the `ParallelOptions` class like so:

```csharp
var nums = Enumerable.Range(0, 100).ToArray();

await Parallel.ForEachAsync(nums, new ParallelOptions { MaxDegreeOfParallelism = 3 }, async (i, token) =>
{
    Console.WriteLine($"Starting iteration {i}");
    await Task.Delay(1000, token);
    Console.WriteLine($"Finishing iteration {i}");
});
```

Here, the max DoP is 3, meaning only 3 invocations of our lambda will be running at once.

There are other properties in the options class, but that's not my focus. Let's get to the meat and potatoes:

## How does it work?

For the rest of this post, I'll be referencing the [.NET Source Browser](https://source.dot.net).

[This is the implementation](https://source.dot.net/#System.Threading.Tasks.Parallel/System/Threading/Tasks/Parallel.ForEachAsync.cs,88) of `ForEachAsync`. There is another implementation and set of overloads dealing with `IAsyncEnumberable`, but the only difference is how it's disposed.

The first thing this method does is do some basic validation, then [sets up a `taskBody` variable of `Func<object, Task>`].

This func sets up the actual work to do, let's step through it.

A `SyncForEachAsyncState<TSource>` object is set up, which is just a bag of state for managing operations.

```csharp
var state = (SyncForEachAsyncState<TSource>)o;
```

We then loop while the state's `CancellationTokenSource` is still valid.

We lock the state and grab the next item in the collection we're iterating over.

```csharp
TSource element;
lock (state)
{
    if (!state.Enumerator.MoveNext())
    {
        break;
    }

    element = state.Enumerator.Current;
}
```

This lock is important, as contention for items in the collection.

We then, if we haven't launched a worker (we'll get to workers in the next section), launch a worker.

```csharp
if (!launchedNext)
{
    launchedNext = true;
    state.QueueWorkerIfDopAvailable();
}
```

We only do this once because each worker is responsible for the launching the next one, and the method it calls is not thread-safe itself.

We then call `LoopBody` on the `state`.

```csharp
await state.LoopBody(element, state.Cancellation.Token);
```

This is just a delegate representing the one passed in as `body` to the `ForEachAsync` method This is where your body is called, and the item from the collection for this iteration is passed in, and matches the `Func<IEnumerable<TSource>, CancellationToken, ValueTask>` signature on the lambda.

The `finally` block is also important here, as it does cleanup on the state and marks it complete so control can be handled back to the caller.

```csharp
 finally
{
    // If we're the last worker to complete, clean up and complete the operation.
    if (state.SignalWorkerCompletedIterating())
    {
        try
        {
            state.Dispose();
        }
        catch (Exception e)
        {
            state.RecordException(e);
        }

        // Finally, complete the task returned to the ForEachAsync caller.
        // This must be the very last thing done.
        state.Complete();
    }
}
```

Pulling back, the final thing thing we do is kick off the process, by constructing our state and queuing the first worker:

```csharp
var state = new SyncForEachAsyncState<TSource>(source, taskBody, dop, scheduler, cancellationToken, body);
state.QueueWorkerIfDopAvailable();
return state.Task;
```

Since our `taskBody` Func always kicks off the next worker, this is all that needs to be called and our Func will handle the rest.

### Queuing workers

The `SyncForEachAsyncState` class has (in its base class) [the `QueueWorkerIfDopAvailable()` method](https://source.dot.net/#System.Threading.Tasks.Parallel/System/Threading/Tasks/Parallel.ForEachAsync.cs,420).

`Dop` in the method name is the degree of parallelism (DoP) we discussed above. It will queue up a worker by calling either [`ThreadPool.UnsafeQueueUserWorkItem()`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadpool.unsafequeueuserworkitem?view=net-6.0#System_Threading_ThreadPool_UnsafeQueueUserWorkItem_System_Threading_IThreadPoolWorkItem_System_Boolean_) or [`Task.Factory.StartNew(Action, CancellationToken, TaskCreationOptions, TaskScheduler)`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskfactory.startnew?view=net-6.0#System_Threading_Tasks_TaskFactory_StartNew_System_Action_System_Threading_CancellationToken_System_Threading_Tasks_TaskCreationOptions_System_Threading_Tasks_TaskScheduler_), but only if the DoP has available room.

Let's take a look.

First, after checking to see if we have remaining DoP, we decrement the DoP.

```csharp
if (_remainingDop > 0)
{
    _remainingDop--;
```

After that, we increment `_completionRefCount`. This actually counts the number of workers running, so I'm not sure why it's named `completion`.

Above, in our finally block in `ForEachAsync`, we had this line:

```csharp
if (state.SignalWorkerCompletedIterating())
```

This method decrements the `_completionRefCount` and if it's now 0, knows that all of our workers are completed.

Finally then start the worker.

```csharp
Interlocked.Increment(ref _completionRefCount);
if (_scheduler == TaskScheduler.Default)
{
    // If the scheduler is the default, we can avoid the overhead of the StartNew Task by just queueing
    // this state object as the work item.
    ThreadPool.UnsafeQueueUserWorkItem(this, preferLocal: false);
}
else
{
    // We're targeting a non-default TaskScheduler, so queue the task body to it.
    Task.Factory.StartNew(_taskBody!, this, default(CancellationToken), TaskCreationOptions.DenyChildAttach, _scheduler);
}
```

## Conclusion

In this post, we saw that the new `Parallel.ForEachAsync()` method is quite complex, and is a great way to safely do async processes on a collection.

We also saw how it manages the complexity of this operation.

 Hopefully, this will help you understand what is going on under the hood.
