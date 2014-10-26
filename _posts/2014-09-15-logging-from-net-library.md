---
layout: post
title: The nightmare that is logging from libraries (in .NET)
excerpt: "One developer's adventures trying to sanely log from a third-party .NET library"
modified: 2014-10-26
#tags: [intro, beginner, jekyll, tutorial]
comments: true
image:
  feature: texture-feature-02.jpg
  credit: Texture Lovers
  creditlink: http://texturelovers.com
---

## The Problem

I work on [Npgsql](http://www.github.com/npgsql/npgsql), the .NET provider for the Postgresql database.
For our upcoming v3.0, I thought I'd revamp the way Npgsql logs messages; it previously had a minimal,
hand-rolled logging class but I thought it would be nice to have all the features of full-fledged
logging frameworks such as [NLog](http://nlog-project.org), [log4net](http://logging.apache.org/log4net), etc.

Sounds great, right? Not quite.

## Common.Logging

It's not a very good idea to use frameworks like NLog and log4net from a
*library*, since you're imposing them on whoever is using you; they might
have chosen some other framework.
[Common.Logging](http://commons.apache.org/proper/commons-logging/) attempt
to be a solution for this. Originating from the Java universe, it's a
relatively simple abstraction layer which you use in your library.
Your users then set it up with the with the specific framework they want
(NLog, log4net), and your logs flow to that.

Sounds great, right? Not quite.

Common.Logging is a dependency for your library. For some inexplicable reason
I can't stomach this idea; maybe it's related to the fact that it brings in
not just one DLL but two; it feels heavy and reeks of Java-ness; all I wanted
was a minimal, clean way to log from my library.

You can find similar comments by
[Daniel Cazzuilnu](http://blogs.clariusconsulting.net/kzu/tracer-the-unified-dead-simple-api-for-all-logging-frameworks-in-existence/),
who wrote a logging abstraction of his own called Tracer. I thought that
might be the solution to my problem, until I understood that it's meant to
work within a single project/solution, and isn't very suitable for a 3rd-party
library wanting to provide logging to any user. See my comments there for details.

## System.Diagnostics

.NET actually comes with its own logging/tracing framework in
System.Diagnostics. No external dependencies, nothing - anyone can use it.

Sounds great, right? Not quite.

System.Diagnostics is, without a doubt, the smelliest logging API ever
conceived. As a general rule, the .NET BCL provides excellent APIs, but
somebody really screwed the pooch when it comes to logging. Even the "newer"
[TraceSource](http://msdn.microsoft.com/en-us/library/system.diagnostics.tracesource(v=vs.110).aspx)
classes introduced in .NET 2.0 don't provide something worth of being called
logging.

Here are some gripes:

* No support for exception logging. You want to see an exception stack trace?
  Format it yourself.
* No logger hierarchy. In sane logging land, different classes or components
  in your program get their own loggers, and these are arranged in a hierarchy.
  The user setting up their logging config can specify which components get
  logged how and where, and they can do that for entire parts of the hierarchy.
  Not so in System.Diagnostics, where if you want to define a single behavior
  for all the TraceSources (=loggers), you have to duplicate config fragments
  for each and everyone one of them (in App.config XML of course, because
  XML is the greatest).
  There is a hack
* No real programmatic configuration access. You're supposed to configure
  TraceSources in App.config. This is usually OK, but sometimes you want to
  modify things programmatically. What if I want to accept some command-line
  switch that changes the logging config? Well, there is no standard way to
  access TraceSources from code. They are managed internally somewhere in the
  static belly of the TraceSource class, and you're out of luck if you need to
  look them up by name.
* No provisions for performance-sensitive logging. In some cases the
  preparation of a log message can be expensive, so you want to check if DEBUG
  is enabled rather than cook up the message just to have it thrown away
  later. They just forgot about that one.

And to top it all off, the mono implementation appears to have some severe issues
([TraceSource.TraceInformation()](http://msdn.microsoft.com/en-us/library/system.diagnostics.tracesource.traceinformation(v=vs.110).aspx) 
doesn't seem to work, Verbose messages are emitted when the switch is set at Information...).

## Summary

So it is 2014 and there seems to be nothing out there that quite solves the problem of lightweight,
simple logging from a library.

I'll be posting a follow-up with the minimal-horror solution adopted for Npgsql...
