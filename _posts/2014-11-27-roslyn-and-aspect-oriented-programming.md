---
layout: post
title: Roslyn and Aspect-Oriented Programming
modified: 2014-11-27
comments: true
---

Microsoft has recently [announced open-sourcing .NET Core](http://blogs.msdn.com/b/dotnet/archive/2014/11/12/net-core-is-open-source.aspx),
which will eventually lead to open-sourcing the entire .NET Framework. The next version of .NET, which is not far off now, will use the new
Roslyn compiler (also open-source), which opens up the compilation process for integration and consumption. Rather than a black box that
reads C# and produces IL, the compiler now exposes the syntax tree and semantic model derived from source code, making it easy to create
tools which understand and modify source code.

Right now, Roslyn's most visible features seem to be Visual Studio quick fixes and refactors - UI-accessible operations that allow code
standardization in your company, etc. But the syntactic/semantic richness exposed by Roslyn seem ideal for another usage type - aspect
oriented programming. AOP loosely refers to implementing cross-cutting programming concerns, usually by applying certain transformations
to your code as part of the build process. For example, you can automatically wrap certain methods with logging directives for tracing
entry/exit without actually writing the code for every method. Tools such as [PostSharp](http://www.postsharp.net/) perform this very well.

However, the code rewriting required for AOP normally happens after compilation - at the IL level. This makes it complicated to write AOP
tools, since manipulating IL isn't easy (note that you can use PostSharp as an AOP platform for your own custom rewriting). But it also
limits the types of rewrites you can do; as far as I know AOP tools are limited to wrapping code *around* your method (much like a Python
decorator), and don't attempt to touch anything in the method code itself. Roslyn can be viewed as an opportunity to take AOP to a whole
new level, rewriting source code rather than IL and doing it anywhere.

An example. Say I have a project which logs messages with some popular logging framework. Some code paths are extremely
performance-sensitive, and so I want these logging invocations to be removed in release builds (but not in debug builds). This can
currently be achieved by wrapping the logging invocation by a method with the
[Conditional](http://msdn.microsoft.com/en-us/library/system.diagnostics.conditionalattribute%28v=vs.110%29.aspx) attribute - the compiler
leaves the invocation out of when compiling without DEBUG. However, with Roslyn I could easily mark methods (or classes) with a
[PerformanceCritical] attribute, and use Roslyn to rewrite logging invocations out of these methods. This approach has the advantage of
removing the needless conditional wrapper, but also of letting me, the project writer, decide exactly where and when to omit the
invocation; the [Conditional] attribute makes the decision on the method invoked rather than per-call side. Note how a *compiler* feature,
the [Conditional] attribute, is superceded by source-level rewriting.

Another good example is [Code Contracts](http://research.microsoft.com/en-us/projects/contracts/). Code contracts allow you to express
conditions and invariants in code, which can be enforced (or not) in runtime or analyzed statically. Runtime enforcing of contracts is
another form of rewriting: declarative constructs such as "Contract.Requires(x > 3)" get rewritten into code which checks the condition
and throws an exception if it isn't met. Doing the rewriting at the source level would probably simplify that process considerably.

So what's needed to actually make Roslyn AOP a usable reality? Not much. The main thing missing is good build integration: it should be
possible, in the project's csproj, to specify rewriter(s) that automatically get invoked on each build prior to compilation. Care must
be taken so that when debugging the original, pre-transformed source code is traversable with as little interference as possible. Concerns
such are these already had to be solved for IL-based rewriters like PostSharp, so they shouldn't be too hard to implement; but hopefully
the .NET team itself will step up and include rewriting at the build-level.

Roslyn opens some really fascinating possibilities in programming, and gives programmers unprecendented ways of parsing their own code
and tweaking it; it makes sense for AOP and code rewriting in general to benefit from it.
