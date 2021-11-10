---
title: A tour of logging in .NET
publishDate: 2021-12-12T00:00:00+00:00
---

I've been thinking a lot about logging this year, as [previous posts will tell you](/posts/logging-ideas).

I decided the next step is to see what libraries are out there and compare them.

> NOTE: This is not a comprehensive review of ALL features. Logging libraries have different features and characteristics, and as always, YMMV and use what's best for your specific needs.

## Features to Compare

These are the absolute baseline for me for logging, in order of importance (to me):

1. Speed
1. Ease of Use
1. Integration with ASP.NET Core
1. Configuration
1. Output methods

## Libraries I'm looking at

* [Microsoft.Extensions.Logging](https://www.nuget.org/packages/Microsoft.Extensions.Logging)
* [Serilog](https://serilog.net/)
* [log4net](https://logging.apache.org/log4net/)
* [NLog](https://nlog-project.org/)

If you know of ones I've missed, let me know!

Note that I've included Microsoft.Extensions.Logging because it does have a default Console output (among others).

## Speed
