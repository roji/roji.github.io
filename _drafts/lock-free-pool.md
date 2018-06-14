---
title: Lock-Free Database Pool
permalink: /lock-free-pool
modified: 2018-02-24
comments: true
---
# Npgsql - a high-performance database driver

Version 4.0 of Npgsql, the PostgreSQL driver for .NET, was released 6 weeks ago, at the same time as the .NET Core 2.1. Both releases put a strong focus on performance, with some very impressive results: running the TechEmpower fortunes benchmark, which has a webapp executing queries against a database, the .NET stack (Linux/ASP.NET Core/PostgreSQL/Npgsql) rose from 124th place in round 15 (only 4 months earlier!) to *7th place* in round 16, making it one of the absolute fastest web/db stacks available. For more details on this, including some numbers, [see this blog post]({% post_url 2018-06-16-npgsql-techempower %}).

A significant part of the achievement comes from improvements in .NET Core itself - the sockets layer was rewritten for performance, and a general focus on perf has improved many other areas. But the database driver, Npgsql, also deserves some of the credit - a lot of effort has been put into making the driver faster. One of the main optimizations in Npgsql 4.0 is a *lock-free connection pool*, which considerably reduced contention and allowed the stack to truly shine.

This post explores some of the specific requirements of a database connection pool, and introduces some basic techniques of lock-free programming which were used in the implementation. The actual code for Npgsql's new lock-free pool [can be found here](https://github.com/npgsql/Npgsql/blob/v4.0.0/src/Npgsql/ConnectorPool.cs), although it's not the easiest piece of code to grok.

I'd like to thank Stephen Toub, Sébastien Ros, Andrew Peters, Arthur Vickers and Diego Viego at Microsoft for the support they provided this effort.

# A basic database connection pool

Pooling sounds like a simple problem, and in many cases it can be. The goal of pooling is to prevent creating new expensive objects by recycling them; at its simplest, a pool is a simple data structure (e.g. an array) which contains idle objects, which we can allocate (or rent), use and then release back into the pool. In the case of database application, the poolable resource is physical connection to the database. Establishing a new physical connection is incredibly expensive: apart from network roundtrips required to establish the connection, in the PostgreSQL case a new process is launched server-side for each new connection; if you do this for each incoming HTTP connection your application will never take off the ground. 

Pooling sounds like a simple problem, and in many cases it can be. The goal is prevent the continuous creation of expensive new objects by recycling them; at its simplest, a pool is a simple data structure (e.g. an array) which contains idle objects, which we can allocate (or rent), use and then release back into the pool. In the case of database application, the poolable resource is physical connection to the database. Establishing a new physical connection is incredibly expensive: apart from the network roundtrips usually required to establish the connection, in the PostgreSQL case a new process is launched server-side for each new connection; if you do this for each incoming HTTP connection your application will never take off the ground.

Now, let's go back to our optimization process. Early in the Npgsql optimization phase, a clear phenomenon was observed: while running intensive benchmarks, the CPU utilization on the benchmark machine never rose beyond 91%. This can point to the network bandwidth being saturated, or to resource saturation on the PostgreSQL server side (running on another machine, of course) - but neither of these was the case. Another possible cause for CPU under-utilization is contention: the different threads are blocking on a lock somewhere. In Npgsql (and probably many other drivers), the pool is the only point where thread synchronization needs to be addressed: once rented, a connection is the sole responsibility of its allocator and isn't threadsafe. This concentrates the difficult problem of synchronization in a single point in the entire driver - which is a good thing - and allowed us to focus on the pool as the likely perf-reducing culprit.

# Lock-free programming

If you suspect lock contention to be an issue, what are your options? The first thing to do is to try to reduce the scope of your locks: the less code executed while taking the lock, the less contention is created. But Npgsql's pool was already pretty optimized in this respect, with locks held for very short spans of code (and never during lengthy I/O operations). This calls for a more extreme solution: lock-free programming.

Lock-free programming means that no locks are taken, at any point, at any time - operations are designed so that they can always interleave with one another without interfering. In practice, this usually means resorting to "interlocked" operations such as compare-and-exchange (also known as compare-and-swap or CAS), which can atomically perform more than one primitive operation. CAS allows you to check if a variable contains some value, and if so, set it to a new value; these two operations are performed atomically, and you also can tell whether a swap was performed or not. Conceptually this is similar to a very short-lived lock around a compare and an assignment, but since these operations are implemented as CPU instructions on all modern architectures, they are very fast and cause an absolute minimum amount of contention.

Before you rush and write any lock-free code, be aware that it's an advanced and very tricky programming technique. At an old office where I used to work, we had a line drawn above the doorframe with the text "you must be this tall to write multithreaded code". Well, lock-free multithreaded code should add a second line significantly above that - it's notoriously hard to get it exactly right, and locks are frequently good enough (while being much less complicated). You get the point. On the other hand, if you're convinced this is a totally theoretical optimization that's never of any use in the real world, that's untrue: as the number of cores increases - with some servers now containing [a whopping 2x32 physical cores](https://www.anandtech.com/show/12694/assessing-cavium-thunderx2-arm-server-reality/12) - avoiding contention becomes a critical concern for some high-perf applications.

But enough words. Here's what a simple, lock-free connection pool could look like. Note that since database connections are expensive, an application isn't expected to have a lot of them, so assume that `PoolSize` below is around 100:

```c#
Connection[] _idle = new Connection[PoolSize];

public Connection Rent()
{
    for (var i = 0; _idle.Length; i++)
    {
        // If the slot contains null, move to the next slot.
        if (_idle[i] == null)
            continue;

        // If we saw a connector in this slot, atomically exchange it with a null.
        // Either we get a connector out which we can use, or we get null because
        // someone has taken it in the meantime. Either way put a null in its place.
        var conn = Interlocked.Exchange(ref _idle[i], null);
        if (conn != null)
            return conn;
    }
    // No idle connection was found, create a new one
    return new Connection(...);
}
```

This code loops over our idle list, and makes use of `Interlocked.Exchange()` to atomically extract a value from an array slot and put a null in its place. This ensures, for example, that two concurrent `Rent()` attempts will never return the same connection. 

```c#
public void Return(Connection conn)
{
    for (var i = 0; i < _idle.Length; i++)
        if (Interlocked.CompareExchange(ref _idle[i], conn, null) == null)
            return;

    // No free slot was found, close the connection...
    conn.Close();
}
```

Returning a connection is slightly more complicated: we use `Interlocked.CompareAndExchange()` to check if a given slot contains null, and only if it does, to assign the given connection to it. All this occurs atomically, and we also get the original contents of the cell as a return value. If we get `null`, we know that the operation succeeded, the connection is safely back in the idle list, and we can exist the function.

# Simple requirements are no fun

Unfortunately, things aren't that simple, and database connection pools have some other pesky requirements which create lots of headaches. .NET's database API abstraction, ADO.NET, doesn't have pooling as a first-class concern, unlike Java's JDBC, for example, which specifies an interface for connection pools. In ADO.NET, the pool is purely an internal implementation detail which database drivers are expected to implement ([see this issue](https://github.com/dotnet/corefx/issues/26714)). Nevertheless, several pool requirements and guarantees have emerged across driver:

1. The pool supports a "maximum pool size" parameter, and ensures that there are never more than that number of physical connections in existence. This is quite important since a database server can only support so many connections, and the same server can be accessed from multiple machines. So it's usually important for the number of connections to be capped in a 100% reliable way. Note that this requirement doesn't usually exist for lighter pooled objects which exist only in memory - it's OK to not have a hard limit on the number of managed resources.
2. If a connection attempt is made but the maximum pool size has already been reached, the attempt needs to wait until a connection is returned to the pool (optionally timing out at some point). This is, again, because we can't go above the defined max pool size, so short of throwing an exception we must wait.
3. Connection attempts must be managed in FIFO order. That is, a connection attempt which starts waiting earlier must be released earlier as well, otherwise we'd have attempts timing out as released connections are handed out randomly (this is how the Npgsql pool used to behave eons ago). This means that a simple lock or `SemaphoreSlim` aren't suitable, since they don't guarantee ordering.

You don't have to be a rocket scientist to understand that we need to add a queue alongside our blissfully simple idle list above. If there are no idle connectors and we have reached the maximum pool size, we enqueue a connection attempt (represented by a `TaskCompletionSource`) into the waiting list, and await on it. On the returning side, we first check the queue to release any waiters, and only put a connection back in the idle list if there aren't any. Now, as long as you're willing to put a big ol' lock around rent/return operations, as Npgsql's pool previously did, this is all fairly simple to implement. Unfortunately, managing these two distinct data structures in a lock-free way is far, far more complicated.

# Managing multiple state variables atomically

Our pool requires us to maintain some state counters: how many connectors are idle, how many are busy (rented), and how many waiters we have; these will be consulted and updated as we rent and release connectors. These three counters must be updated atomically: as we hand out an idle connector a someone, we must both decrement the idle counter and increment the busy counter - and we want to do this without locking. Since interlocked operations only work on a a single value (int, long), you would think they're unusable in this situation. Fortunately, C# allows us to use structs with explicit layout to reference the same chunk of memory in two different ways, once as a several small values, and once as a single large value. Consider the following code:

```c#
[StructLayout(LayoutKind.Explicit)]
internal struct PoolState
{
    [FieldOffset(0)]
    internal short Idle;
    [FieldOffset(2)]
    internal short Busy;
    [FieldOffset(4)]
    internal short Waiting;
    [FieldOffset(0)]
    internal long All;
}
```

The `LayoutKind.Explicit` value tells C# that we will be manually laying out the fields in the struct, by specifying a byte offset for each one. The first three fields are straight-forward: we have three short counters, taking up 2 bytes each. So far so good. But the fourth field is an 8-byte long that is defined at offset 0 - over the same memory occupied by the first 3; this is referred to as an undiscriminated union. The key point here is that we can modify the individual counters, and then use an interlocked operation on the long to write *all* the changes to some other chunk of memory, atomically:

```c#
// Create a local copy of the pool state, in the stack
PoolState newState;
newState.All = Volatile.Read(ref _state.All);   // Ignore Volatile.Read() for now
// Modify it
newState.Idle--;
newState.Busy++;
// Commit the changes
Interlocked.Exchange(ref _state.All, newState.All);
```

While regular field access happens via `Idle` and `Busy` as usual, the "I/O" of the struct happens through the `All` field, with the help of interlocked operations which guarantee atomicity.

# But what about concurrency?

The whole point of lock-free code is to work well in concurrent scenarios. Those used to thinking concurrently have already spotted the problem with the above code: it's possible that after we copied `_state` and as we're preparing our changes, someone else has modified `_state`, changing the counters. When we use `Interlocked.Exchange()`, we're overwriting any such changes, corrupting the pool state and dooming us to hours of agonizing concurrency debugging. The solution is to make sure that the pool's state hasn't changed from the time we copied it (or more precisely, that the two states are equal). We can use `Interlocked.CompareAndExchange()` to do this:

```c#
var sw = new SpinWait();
while (true)
{
    // Create a local copy of the pool state, in the stack
    PoolState oldState;
    oldState.All = Volatile.Read(ref _state.All);
    var newState = oldState;
    // Modify it
    newState.Idle--;
    newState.Busy++;
    // Try to commit the changes
    if (Interlocked.CompareAndExchange(ref _state.All, newState.All, oldState.All) == oldState.All)
        break;
    // Our attempt to increment the busy count failed, Loop again and retry.
    sw.SpinOnce();
}
```

This is code is a bit harder to grok. We first create a local copy of `_state`, called `oldState`. We then create a copy of *that* copy, `newState`, where we will update our counters. Finally, the `Interlocked.CompareAndExchange()` will write `newState`, but only if `_state` hasn't been changed in the meantime: it must be equal to `oldState`, which is our snapshot of the state. If all went well, the operation returns the old state and we happily break out of the loop. If, however, the operation failed, we delay a little bit (that's what `SpinWait.SpinOnce()` is for), and then retry to entire operation, copying the new pool's state and attempting to update it. Eventually this will succeed.

This is what lock-free programming frequently looks like; rather than locking, we loop, trying to update state via interlocked operations, until we eventually succeed.

{% comment %}

# Lock-free programming

* Does not matter to everyone - low-latency, query results in memory, etc.
# Other providers

* Copy-paste, make available to others, long-term introduce a pooling API like JDBC
* Npgsql vs. pgbouncer/pgpool
* Microsoft month

TODO


# DEAD CRAP

An implementation of a pool allocation operation can be roughly the following:

1. First check an "idle list" to see if an idle connection exists. If so, take it out and return it.
2. If no idle connections are available, check the total number of connections (both idle and busy). If the total number is less than max pool size, create a new connection and return it
3. If there are no idle connectors and we've reached the max pool size, we're out of luck and must wait until someone releases a connection. Enqueue a connection attempt (represented as a `TaskCompletionSource`) in a queue and await it. The next release operation will dequeue and complete our `TaskCompletionSource`, allowing us to return.

Releasing a connection involves first checking if there are any waiters in the queue, and if not, placing the connection back in the idle list.

Because of the complexity of the above and the management of two different data structures, Npgsql 3.2 had a simple lock on each pool. The lock was never taken while I/O was being performed (e.g. opening a new connection), but it still caused quite a bit of contention. Enter lock-free programming

{% endcomment %}
