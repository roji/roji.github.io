---
title: Npgsql 4.0 - A High Performance Database Driver
permalink: /npgsql-4-perf
modified: 2018-06-16
comments: true
---
Npgsql 4.0 was finally released on May 30th, and [as the release notes mention](http://www.npgsql.org/doc/release-notes/4.0.html), a big focus of that release was making the driver as fast as it could be.

The release almost coincided with [round 16 of the TechEmpower benchmarks](https://www.techempower.com/blog/2018/06/06/framework-benchmarks-round-16/). TechEmpower is a set of standardized benchmarks which measure various web scenarios on different platforms, languages, frameworks and databases; it has been an important way to pit technology stacks against one another and to shine a light on performance bottlenecks. In [round 13](https://www.techempower.com/blog/2016/11/16/framework-benchmarks-round-13/), whose results were published in November 2016, the newly-released .NET Core and ASP.NET made a splash by making very significant improvements in the plaintext benchmark, making .NET a real contender as a high-performance web stack. However, the plaintext benchmark measures pure framework speed and does not include a database, making it somewhat non-realistic as a gauge for actual web performance. In the fortunes benchmarks, which exercises database access and is much closer to a real-world webapp, ASP.NET Core [only got to 124th place in round 15](https://www.techempower.com/benchmarks/#section=data-r15&hw=ph&test=fortune).

Round 16 ran on .NET Core and ASP.NET Core 2.1 (rc1), which made a big push in terms of performance. Many optimizations were made at the .NET Core level, and the managed sockets layer (which Npgsql uses) received some special attention. The new socket layer performed so well that ASP.NET Core 2.1 uses it by default, abandoning libuv as its default transport option. But Npgsql also made a very significant progress on performance, thanks to an intensive work cycle which included collaboration with people inside Microsoft.

All this work paid off: [.NET rose to *7th place*](https://www.techempower.com/benchmarks/#section=data-r16) in the fortunes benchmark in round 16, up from 124th place in round 15. Since some of the top results include some obscure and/or highly specialized web frameworks, this is all the more impressive, making .NET one of the very highest-performing web stacks out there. The following table shows performance running the fortunes webapp, breaking down the increase between Npgsql 4.0 and .NET Core/ASP.NET 2.1:

OS       | .NET Core | Npgsql 3.2 RPS | Npgsql 4.0 RPS |
---------|-----------|----------------|----------------|
Windows  | 2.0       | 71,810         | 107,400        |
Windows  | 2.1       | 101,546        | 132,778        |
Linux    | 2.0       | 73,023         | 93,668         |
Linux    | 2.1       | 78,301         | 107,437        |

The benchmarks were all run on an HP z440 with an E5-1650 v3 @ 3.50 GHz w/ 15MB Cache, 6 Cores / 12 Threads, and 32GB DDR4-2133.

On Windows, this table shows a *50% perf improvement between Npgsql 3.2 and 4.0*, running on .NET Core 2.0. Switching to .NET Core 2.1 adds another 24%, bringing the total improvement to 85% - quite impressive indeed. The Linux figures are lower, giving reason to believe that the some aspects of .NET Core may be under-optimized on Linux. Since the TechEmpower results are on Linux, this could mean that the .NET stack could go even higher in the rankings; hopefully this will happen in future versions.

I intend to write a post or two about the actual optimizations done in Npgsql to make this happen - a description of Npgsql's new lock-free connection pool should be up soon. I'd like to thank Sebastien Ros at Microsoft for running the benchmarks and providing the above figures, and also to Stephen Toub, Andrew Peters, Arthur Vickers and Diego Vega for their valuable help and advice.

It should be noted that some of the perf improvements done for Npgsql 4.0 can also be carried over to other database drivers. I'm in touch with @bgrainger, the creator of MySqlConnector, an open source alternative to Oracle's MySQL driver, which is already the 4th MySQL entrant in round 16's fortunes. There are also some conversations on how to improve SqlClient performance.

