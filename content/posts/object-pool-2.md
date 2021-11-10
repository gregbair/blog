---
title: Object Pool - Proxying calls
tags:
- object-pool
- csharp
- patterns
slug: object-pool-proxy
date: 2020-10-21T11:30:56+00:00
---

> **Note:** This is the second post in a [series](/tags/object-pool) about an object pooling pattern. You can follow along with code in this [repo on Github](https://github.com/gregbair/object-pool). The repo will be steps ahead of this series, so don't worry if there's things in there I haven't explained.

- [Dynamic Proxy](#dynamic-proxy)
- [Why a Dynamic Proxy](#why-a-dynamic-proxy)
- [Proxy generator](#proxy-generator)
- [Wrapper](#wrapper)
- [Combining the wrapper and the interceptor](#combining-the-wrapper-and-the-interceptor)
- [Summary](#summary)

## Dynamic Proxy

For our proxy object, we're going to use [Dynamic Proxy](https://www.castleproject.org/projects/dynamicproxy/) from the Castle Project. This package allows us to create a pass-through object for any type that implements the interface we're going to be pooling.

Dynamic proxying is a form of the [decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern) that doesn't care what the underlying object is. Since C# is a strongly-typed language, it's burdensome to create a proxy that can handle any type - you'd need to know at compile time (when you write your code) what type you're proxying.

Dynamic proxying hijacks the runtime typing, using reflection to create stub and proxy classes at runtime to allow more flexibility in situations like ours where we want to work with types that are only known at runtime.

We're going to pass through any method call except one - `Dispose()`. We'll use that to return our object to the pool.

## Why a Dynamic Proxy

If you know the type of object you're pooling ahead of time, you don't need to use dynamic proxying. You can just manually create a proxy object that implements the same interface, and pass through all the calls you need to, and not pass through the ones you don't. For our pool, though, we're trying to make something more general so that it's reusable.

## Proxy generator

The dynamic proxy mechanism from the Castle Project uses what's called a proxy generator. In short, we define an interceptor that passes through all the method calls (except `Dispose`), and use that to generate at runtime a proxy object. Note that this does incur a small bit of overhead, so if you're only pooling one type, and know that type, you can skip doing this.

## Wrapper

We also need something to wrap our object and put some metadata on it that will be  useful to the pool. Attributes like which pool it belongs to, a unique ID, and the target object are the metadata we're storing for now. Later we might add when the object was created if we need to dispose "stale" objects.

## Combining the wrapper and the interceptor

For simplicity, we can combine both of these concepts into a single class, called our [`PooledObjectWrapper<TObject>`](https://github.com/gregbair/lagoon/blob/main/src/Lagoon/PooledObjectWrapper.cs#L13).

As part of its implementaiton of `IInterceptor`, we have a method `Intercept(IInvocation invocation)` that we'll use to intercept method calls. Let's take a look at that method below:

```c#
// IInvocation is an object passed in by Castle Project containing metadata
// about the method call.
public void Intercept(IInvocation invocation)
{
    if (invocation is null)
    {
        throw new ArgumentNullException(nameof(invocation));
    }

    if (!invocation.Method.Name.Equals("Dispose", StringComparison.OrdinalIgnoreCase))
    {
        // We only want to intercept the Dispose method.
        invocation.Proceed();
    }
    else
    {
        // If Dispose is called, return this object to the pool.
        _pool.ReturnObject(this);
    }
}
```

Because of the way we are intercepting the `Dispose` method, we can hijack it to not actually dispose the object, but instead return it to our pool.

Our interceptor/wrapper also has its own dispose method, which will be called by the pool in its pruning operation (we'll cover that later). It actually disposes the wrapped object.

## Summary

We looked at how dynamic proxies work, and introduced an interceptor and a wrapper for our object.

For further info about dynamic proxies and the interceptor, see the Castle Project's [docs](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy.md).

Next, we'll look into how the pool manages retrieving, creating, activating, and disposing objects.
