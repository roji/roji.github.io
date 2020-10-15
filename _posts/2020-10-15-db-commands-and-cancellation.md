---
title: The Curious Case of Commands and Cancellation
permalink: /db-commands-and-cancellation
modified: 2020-10-15
comments: true
---
Async brought a world of goodness (and complexity) to .NET, including the concept of cancellation: since async operations are by their nature supposed to take a while, it makes sense to allow us to cancel mid-way and exit early. Like async in general, cancellation took some time to propagate everywhere - socket operations only started honoring cancellation token in [.NET Core 3.0](https://github.com/dotnet/runtime/issues/23736). For the upcoming 5.0 release of Npgsql, the PostgreSQL database driver, a lot of work is going on to provide a good command cancellation story (thanks [@vonzshik](https://github.com/vonzshik)!!), and it is far more complicated than you'd think.

[DbCommand](https://docs.microsoft.com/dotnet/api/system.data.common.dbcommand) is the standard .NET type which represents a query you run against a database - you set SQL and parameters on it, invoke [ExecuteReaderAsync](https://docs.microsoft.com/dotnet/api/system.data.common.dbcommand.executereaderasync), and it gives you back a [DbDataReader](https://docs.microsoft.com/dotnet/api/system.data.common.dbdatareader) which allows you to consume the results:

```c#
await using var cmd = connection.CreateCommand();
cmd.CommandText = "SELECT something_from_the_database";

await using var reader = await cmd.ExecuteReaderAsync();
// The query has been started and is now running in the background
// Consume results via the reader
```

Now, back in the old days - before async was even a thing - DbCommand already had a [Cancel](https://docs.microsoft.com/en-us/dotnet/api/system.data.common.dbcommand.cancel?view=netcore-3.1#System_Data_Common_DbCommand_Cancel) method. This method attempts to cancel the ongoing execution of the query, on a best-effort basis, by doing whatever is appropriate for your database. When async came along, all the database types were retrofitted with async methods accepting cancellation tokens: [DbCommand.ExecuteReaderAsync](https://docs.microsoft.com/dotnet/api/system.data.common.dbcommand.executereaderasync), [DbDataReader.ReadAsync](https://docs.microsoft.com/dotnet/api/system.data.common.dbdatareader.readasync), etc. Logically, invoking the cancellation token is the async analog of calling the old DbCommand.Cancel - both semantically mean the same thing. Or so it would seem.

When you pass a cancellation token to some method, the general expectation is for the token to control that specific invocation; if you trigger the token, that invocation should terminate as early as possible and throw an [OperationCanceledException](https://docs.microsoft.com/dotnet/api/system.operationcanceledexception). The simplest example is [HttpClient.GetAsync](https://docs.microsoft.com/dotnet/api/system.net.http.httpclient.getasync#System_Net_Http_HttpClient_GetAsync_System_String_): that method call represents one potentially long process, and the cancellation token can abort that process; when the method completes, you know nothing lingers in the background. The database API, in contrast, is more complex: when DbCommand.ExecuteReaderAsync completes, the query has only just started, is (likely) still running, and may continue running for a very long time. The DbDataReader it returns allows you to start processing the result stream, possibly in parallel, while the database server is still running the query and sending results back.

So ExecuteReaderAsync starts some background process (the query), which doesn't complete when the method itself completes - why is that significant? One question this raises, is how one goes about cancelling the query *after* ExecuteReaderAsync completes; the traditional DbCommand.Cancel API doesn't have this problem, because it's a method on the DbCommand, rather than a token you pass to some method call.

Another related question is what happens with the various methods on DbDataReader which also accept a cancellation token, such as ReadAsync: what should happen when the token for ReadAsync is triggered? The usual expectation in the async world is again, for the token to only cancel the method to which it was passed, i.e. ReadAsync; but if we do that, we're left with no token-based means to cancel the query at all - which is a pretty important requirement. We could tell users who want to cancel the query to rig their cancellation tokens to call our old DbCommand.Cancel:

```c#
public async Task ExecuteSomethingAsync(DbConnection connection, string sql, CancellationToken cancellationToken = default)
{
    await using var cmd = connection.CreateCommand();
    cmd.CommandText = sql;

    await using var reader = await cmd.ExecuteReaderAsync(cancellationToken);
    using var registration = cancellationToken.Register(() => cmd.Cancel());

    // Process results
}
```

But this would have to be done everywhere where a potentially cancellable command needs to be executed, and isn't very discoverable. Finally, it just doesn't seem incredibly useful to allow ReadAsync to be cancelled while leaving the query itself running; it's simply unlikely that a later retry would produce useful results where an earlier ReadAsync was cancelled. Yes, this business of "detached" async background processes which don't correspond to method calls isn't entirely trivial.

The solution we opted for was to treat the ReadAsync's token - and indeed, all tokens accepted by methods on DbDataReader - in the same way as we treat ExecuteReaderAsync's: triggering it cancels the query. This solves the typical user requirement: if a cancellation token comes from somewhere (rigged to some GUI button, for example), and is passed to all database methods, then the query gets cancelled if it gets triggered.

This does have one peculiar consequence. It is quite standard for async methods to start with the following:

```c#
public async Task SomeLongThing(CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();

    // ... actual stuff ...
}
```

This performs an upfront check on the token, and immediately returns a cancelled Task if the method is invoked with an already-cancelled token. But in our case, calling the method with a cancelled token is actually the way to request cancellation of the query. It's a bit odd, but it works and performs what people generally want.

Hate it? Love it? Let us know.