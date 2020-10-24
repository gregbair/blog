---
title: Object Pool - Proxy
tags:
- object-pool
- csharp
- patterns
slug: object-pool-proxy
date: 2020-10-21T11:30:56+00:00
draft: true
---

> **Note:** This is the second post in a [series](/tags/object-pool) about an object pooling pattern. You can follow along with code in this [repo on Github](https://github.com/gregbair/object-pool). The repo will be steps ahead of this series, so don't worry if there's things in there I haven't explained.

- [Dynamic Proxy](#dynamic-proxy)
- [Why a Dynamic Proxy](#why-a-dynamic-proxy)
- [Proxy generator](#proxy-generator)
  - [Interceptor](#interceptor)
- [Wrapper](#wrapper)
- [Summary](#summary)

## Dynamic Proxy

For our proxy object, we're going to use [Dynamic Proxy](https://www.castleproject.org/projects/dynamicproxy/) from the Castle Project. This package allows us to create a pass-through object for any type that implements interface we're going to be pooling..

We're going to pass through any method call except one - `Dispose()`. We'll use that to return our object to the pool.

## Why a Dynamic Proxy

If you know the type of object you're pooling ahead of time, you don't need to use dynamic proxying. You can just manually create a proxy object that implements the same interface, and pass through all the calls you need to, and not pass through the ones you don't. For our pool, though, we're trying to make something more general so that it's reusable.

## Proxy generator

The dynamic proxy mechanism from the Castle Project uses what's called a proxy generator. In short, we define an interceptor that passes through all the method calls (except `Dispose`), and use that to generate at runtime a proxy object. Note that this does incur a small bit of overhead, so if you're only pooling one type, and know that type, you can skip doing this.

### Interceptor

Our interceptor looks like this:

```csharp
/// <summary>
/// Intercepts calls for <typeparamref name="TProxy"/>.
/// </summary>
/// <typeparam name="TProxy">The type to intercept calls for.</typeparam>
internal class PooledObjectInterceptor<TProxy> : IInterceptor
    where TProxy : class, IDisposable
{
    private readonly IObjectPool<TProxy> _pool;
    private readonly PooledObjectWrapper<TProxy> _wrapper;

    /// <summary>
    /// Initializes a new instance of the <see cref="PooledObjectInterceptor{TProxy}"/> class.
    /// </summary>
    /// <param name="pool">The pool to which this interceptor should return objects to.</param>
    /// <param name="wrapper">The proxy object wrapper.</param>
    public PooledObjectInterceptor(IObjectPool<TProxy> pool, PooledObjectWrapper<TProxy> wrapper)
    {
        _pool = pool ?? throw new ArgumentNullException(nameof(pool));
        _proxy = proxy ?? throw new ArgumentNullException(nameof(proxy));
    }

    /// <inheritdoc/>
    public void Intercept(IInvocation invocation)
    {
        if (invocation is null)
        {
            throw new ArgumentNullException(nameof(invocation));
        }

        if (!invocation.Method.Name.Equals("Dispose", StringComparison.OrdinalIgnoreCase))
        {
            invocation.Proceed();
        }
        else
        {
            _pool.ReturnObject(_wrapper);
        }
    }
}
```

Ignore what we're actually doing when we capture the `Dispose()` method, we'll explain that later.

What we're doing inside the `Intercept` method is inspecting an `IInvocation` object. This object is supplied by Dynamic Proxy and contains the runtime information about the method being called. As you can see, we're capturing the `Dispose()` method and handling that specifically, but every other call, we're calling `invocation.Proceed()` which tells Dynamic Proxy to pass through to the actual object.

To use this generator, you'd call:

```csharp
IFoo foo;
var wrapper = new PooledObjectWrapper<IFoo>(foo);
var generator = new ProxyGenerator();
var objProxy = generator.CreateInterfaceProxyWithTarget(
    obj,
    new PooledObjectInterceptor<IFoo>(this, wrapper);
);
```

At this point, `objProxy` is of type `IFoo`. Also, `this` refers to our pool, which we'll cover in the next post.

Castle Project documents the different proxy target types [here](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy-kinds-of-proxy-objects.md).

## Wrapper

To store meta information about the object, we implement a pseudo-decorator pattern to wrap the actual proxied object. This will give us the ability in the future to store things like an ID, when the object was created, etc. to help maintain our pool. Our pool will store our objects inside this wrapper and query for information from it when making decisions about what objects to keep and which to discard.

```csharp
/// <summary>
/// A proxy to wrap objects of type <typeparamref name="TObject"/>.
/// </summary>
/// <typeparam name="TObject">The type of object to wrap.</typeparam>
public class PooledObjectWrapper<TObject>
    where TObject : class, IDisposable
{
    /// <summary>
    /// Gets the ID of this proxy.
    /// </summary>
    public Guid Id { get; }

    /// <summary>
    /// Gets the actual object that's wrapped in this proxy.
    /// </summary>
    public TObject Actual { get; }

    /// <summary>
    /// Initializes a new instance of the <see cref="PooledObjectWrapper{TObject}"/> class.
    /// </summary>
    /// <param name="actual">The object to be wrapped.</param>
    public PooledObjectWrapper(TObject actual)
    {
        Actual = actual ?? throw new ArgumentNullException(nameof(actual));
        Id = Guid.NewGuid();
    }
}
```

Our `Actual` property is simply the proxied object created by the `ProxyGenerator`.

## Summary

We looked at how dynamic proxies work, and introduced an interceptor and a wrapper for our object.

For further info about dynamic proxies and the interceptor, see the Castle Project's [docs](https://github.com/castleproject/Core/blob/master/docs/dynamicproxy.md).
