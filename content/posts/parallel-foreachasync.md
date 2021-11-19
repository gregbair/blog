---
title: Parallel.ForEachAsync Deep Dive
publishDate: 2021-12-12T00:00:00+00:00
draft: true
---

## Intro

.NET 6 introduced a small feature that's important, but was probably overlooked - [`Parallel.ForEachAsync`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel.foreachasync?view=net-6.0).

This is a (IMO) pretty powerful and elegant implementation of something that's been missing for a while from .NET - parallel processing of async methods.

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

> NOTE: I would not actually use the `ForEachAsync<TSource>(IAsyncEnumerable<TSource>, Func<TSource,CancellationToken,ValueTask>)` overload of this method. I'll show why and better options below.

What this gives you is a basic foreach loop over your collection, in parallel, with safeguards in place to not overwhelm your system (kinda).

In this invocation, we're not specifying a _degree of parallelism_ constraint, meaning we're not saying how many iterations of the lambda to run at once. So what happens? You get the default, which is the number returned by `Environment.ProcessorCount`. That property, though, is misnamed, as it's not the number of processors, but the number of _cores_ your system has. (Mine shows 16, as I have an AMD Ryzen 7, which has 8 physical cores + hyperthreading).

The safer thing to do is to specify a max degree of parallelism. The right number is the classic developer answer "it depends". It depends on your system, the size of your collection, a host of factors that are outside the scope of this post. Your best bet is to make it configurable and test a lot of different values.

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

Here, the max DoP is 3.

There are other properties in the options class, but that's not my focus. Let's get to the meat and potatoes:

## How does it work?