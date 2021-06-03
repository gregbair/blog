---
title: Logging Ideas
tags:
- csharp
- logging
- patterns
slug: logging-ideas
date: 2021-06-03T05:32:00+00:00
---

I've been thinking lately about logging, specifically in C#/.NET. In fact, every few months, I joke around with others at work that I'm going to end up writing my own logging framework.

There are a few things I want from a logging subsytem -

- [Fast](#fast)
- [Highly configurable with sane defaults](#highly-configurable-with-sane-defaults)
- [Well-tested](#well-tested)
- [Documented](#documented)
- [Extensible](#extensible)
- [Benchmarked](#benchmarked)
- [Runtime configurability](#runtime-configurability)
- [Instrumented](#instrumented)
- [Best effort](#best-effort)
- [Integration with .NET logging framework](#integration-with-net-logging-framework)

In this post, I'll expand on these. In future posts, I'll evaluate what's out there with current libraries.

## Fast

I want a logging system to be super fast. I don't want logging to be a bottleneck for my application. This is a drawback I see in many logging systems today.

Usually, you're waiting for I/O. There are ways to speed up I/O in .Net, and for most uses, it's fast enough.

Sometimes, though, developers get a little too happy with logging, and create chatty logs.

My thought is maybe there's a way to move the I/O parts off to background threads to allow a call to `Log()` (or whatever the associated method is). I could be wrong about this, we'll see.

I also need to think about what's an acceptable amount of time for a logging statement to take? Even 1ms seems too slow, because if I'm calling an endpoint in a service, and there's say 50 log statements during that process, that means I've added 50ms of overhead to my time.

## Highly configurable with sane defaults

I want logging to be infinitely tunable. I want to be able to drill down to things like log file rollover retention policies for file-based outputs.

I want to be able to adjust resiliency (timeouts, retries, etc.) for outputs that reach to external services like databases or log aggregators like Splunk.

Something that almost all current libraries offer is log level configuration per "category". That needs to be there. Microsoft logs are extremely chatty, even at the INFO level.

Building on that, I want user-configurable levels. This is sorely missing from current C# options, but is available in other ecosystems like log4j2.

Think about this, what if you need something between debug and verbose? What if you want to define log levels from a business requirements POV (though I'm not sure that's a good idea.)

I also want sane defaults. Default configuration in any software system is usually (somewhat educated) guesswork. I want thought behind configuration and defaults.

## Well-tested

I want lots of really good and relevant tests. Both unit and functional tests. The combinatorics of configurations creates a lot of edge cases. I want those cases tested.

## Documented

I want comprehensive documentation, with:

- Examples
- A cookbook
- API docs

## Extensible

I want to have an extensible API that allows me to basically change every part of the process - ingestion, formatting, output.

## Benchmarked

I want benchmarks. How does the performance of your logging framework compare to "competitors"? What about real-world scenarios?

## Runtime configurability

One thing that's really missing from current libraries is runtime configurability. I need to be able to easily change things like logging levels at runtime.

When something's going wrong in production, I want to change from `WARN` to `DEBUG` easily.

The sticky point here is distributed systems. If I have 3 instances of a service, I want them to (possibly) be able to share that configuration.

## Instrumented

I love metrics and data, and one thing that makes me excited right now is [OpenTelemetry](https://opentelemetry.io/). There's lots of industry investment in this protocol/spec, and .NET has already done this through the [`System.Diagnostics` APIs](https://devblogs.microsoft.com/dotnet/opentelemetry-net-reaches-v1-0/).

I want a logging framework to support this.

## Best effort

With my want for perf and resiliency, I understand that there's tradeoffs. If I use say, a buffer for better perf, I need to accept that if my service dies before that buffer is flushed, I lose some logging statements. I think what I'd prefer here is a way to say "hey, this statement is super-important, it absolutely needs to get processed ASAP" vs. "hey, this is nice-to-know info, get to it when you can".

## Integration with .NET logging framework

This is absolutely critical, and highly controversial.

Microsoft has been criticized in its early forays into open source (even now) about taking ideas the community has and doing the old MS thing of embrace, extend, extinguish. This happened most recently with JSON processing, first embracing [JSON.NET](https://www.newtonsoft.com/jsons), the _de facto_ standard JSON library, hiring its maintainer, then [writing their own version (`System.Text.Json`)](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-overview).

I think with logging, MS struck a nice balance, using its own interfaces with `Microsoft.Extensions.Logging` and a simple implementation and making it somewhat easy for 3rd-party logging providers to integrate into that system.

However, I want to be able to blend a different way of using the logging sytem while also supporting built-in ASP.NET logging. ASP.NET is almost entirely dependent on constructor injection, which I don't hate, but isn't always the best DI strategy, compared to other strategies like property injection or method injection.
