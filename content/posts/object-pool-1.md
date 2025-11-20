---
title: Object Pool - An Introduction
tags:
- object-pool
- csharp
- patterns
slug: object-pool-intro
date: 2020-10-21
---

> **Note:** This is the first post in a [series](/tags/object-pool) about an object pooling pattern. You can follow along with code in this [repo on Github](https://github.com/gregbair/object-pool). The repo will be steps ahead of this series, so don't worry if there's things in there I haven't explained.

- [Overview](#overview)
- [Real world examples](#real-world-examples)
  - [Microsoft SqlClient](#microsoft-sqlclient)
  - [Ldaptive](#ldaptive)
- [Components of a pool](#components-of-a-pool)
  - [Object Pool](#object-pool)
  - [Object](#object)
  - [Proxy](#proxy)
  - [Object Factory](#object-factory)
- [Summary](#summary)

## Overview

An Object pool is a pattern used to maintain a pool of objects (_natch_) that is useful in situations where initialization of those objects is expensive, such as database connections or network connections in general.

## Real world examples

### Microsoft SqlClient

When a new `SqlConnection` is opened, a [connection pool](https://github.com/dotnet/SqlClient/blob/v2.0.1/src/Microsoft.Data.SqlClient/netfx/src/Microsoft/Data/ProviderBase/DbConnectionPool.cs) is either created or used based on the connection string. This also involves pool "groups", which is how the pool-per-connection string is managed, but is out of the scope of this series.

### Ldaptive

Ldaptive is a Java LDAP connection library. Most of the concepts in this series will be adapted from Ldaptive, as it's an accessible but robust [pool implementation](https://github.com/vt-middleware/ldaptive/blob/master/core/src/main/java/org/ldaptive/pool/ConnectionPool.java).

## Components of a pool

![Component Diagram](/object-pool-components/object-pool-components.svg)

### Object Pool

The object pool is the main mechanism for maintaining the objects. It handles scaling up or down, delegation of the creation of new objects, and acts as traffic cop to hold requests for a bit when no new connections are available.

### Object

The object is the thing requiring pooling. We expect this to implement `IDisposable`, as most objects needing pooling contain unmanaged resources. Even if it doesn't, we'll show in a later post how to create a class that does implement `IDisposable` to be usable in our pool.

### Proxy

The proxy wraps the object and mainly tracks the lifecycle of the object in the pool and intercepts calls to `Dispose()` to return the object to the pool. This will be explained in the next post.

### Object Factory

The factory is responsible for actually creating the object, as the pool does not know how to actually construct its members.

## Summary

This post hopefully gave a good intro into the why and what of pooling mechanisms. In the next post, we'll go more in depth into the proxy, including talking about dynamic proxies and introducing [Dyanmic Proxy](https://www.castleproject.org/projects/dynamicproxy/) from the Castle Project.
