---
title: Query parameters, batching and SQL rewriting
permalink: /parameters-batching-and-sql-rewriting
modified: 2021-08-17
comments: true
---
In the upcoming version 6.0 of the Npgsql PostgreSQL driver for .NET, we implemented what I think of as "raw mode" ([#3852](https://github.com/npgsql/npgsql/pull/3852)). In a nutshell, this means that you can now use Npgsql without it doing anything to the SQL you provide it - it will simply send your queries as-is to PostgreSQL, without parsing or rewriting them in any way. Explaining what this means is a great opportunity to go into some interesting aspects of database programming - so let's dive in.

## Parameters

Parameters are important in database programming: instead of putting values directly into your SQL query, you integrate a placeholder which references a parameter value that's delivered separately. This is important for preventing SQL injection attacks, but also helps performance through plan caching and prepared statements. Anybody who's used .NET's database API (ADO.NET) knows how parameters work:

```c#
var cmd = new NpgsqlCommand("SELECT * FROM employees WHERE first_name = @FirstName AND age = @Age", conn)
{
    Parameters =
    {
        new("FirstName", "Shay"),
        new("Age", 18)
    }
};
```

A command has a collection of parameters, and each parameter has a name and a value. Pretty straightforward... or is it?

It turns out that while some databases accept such name/value parameter pairs (e.g. SQL Server), PostgreSQL actually has a positional parameter system! Rather than the named parameter placeholders `@FirstName` and `@Age`, it expects to get `$1` and `$2`, which refer to positions in the parameter list. And indeed - there's quite a zoo of parameter placeholders once you look around: Oracle does have named parameters like SQL Server, but uses a semicolon as the prefix (so `:Age` rather than `@Age`). In ODBC, parameter placeholders are simply question marks, which also bind positionally to parameters (this means that it's impossible to refer to the same parameter twice without sending it twice as well).

What a mess. Now, the ADO.NET documentation calls all this out, [clearly stating](https://docs.microsoft.com/dotnet/framework/data/adonet/configuring-parameters-and-parameter-data-types#working-with-parameter-placeholders) that parameter placeholders vary across data providers. In fact, if you look at the [DbParameterCollection](https://docs.microsoft.com/dotnet/api/system.data.common.dbparametercollection) class, you'll find a collection that is both named like a dictionary (for SQL Server, Oracle...) and ordered like an array (for ODBC, PostgreSQL). But at some prehistoric moment in Npgsql's history, someone made the decision to support named parameters. This was probably done to make it easier to port applications from SQL Server to PostgreSQL, without having to change any SQL - not a bad idea. Unfortunately, that also means that Npgsql has to internally parse your SQL query and rewrite it to send the following to PostgreSQL:

```sql
SELECT * FROM employees WHERE first_name = $1 AND age = $2
```

## Batching

As fascinating as the above mess is, let's leave it for a second and concentrate on something else - statement batching. When you want to execute two unrelated SQL statements, it's far more efficient to send both at the same time, and not wait for the first to complete before sending the second. In principle, any type of SQL statement can be batched in this way: an UPDATE and a DELETE, 5 different SELECTs, anything; if you're not already batching where you could be, I highly recommend giving it a try.

The current way to perform batching with ADO.NET looks like this:

```c#
var cmd = new NpgsqlCommand("SELECT * FROM employees; SELECT * FROM departments", conn);
```

You simply pack two SQL statements into a single command - separated by a semicolon - and execute that command as a single batch. Pretty straightforward... or is it?

The above works as-is on SQL Server, but the situation is a bit more complicated on PostgreSQL. PostgreSQL supports two protocols on the wire: the simple protocol and the extended protocol. The former does allow sending multiple statements as above, but has no support for parameters, prepared statements and various other features. At some point in the past, Npgsql actually used the simple protocol, and got around the lack of parameter support by interpolating parameter values directly into the SQL (client-side binding); this meant Npgsql needed to know how to generate (and parse) string representations of all supported data types, and that's still inefficient due to the lack of real parameterization and prepared statements. Modern Npgsql exclusively uses the extended protocol, where each protocol message corresponds to exactly one SQL statement, with its own parameter list - no semicolons allowed.

So how does the above batching code work? You guessed it! Npgsql parses the SQL, locates the semicolons and breaks up the command's text into multiple extended protocol messages.

## So what's the big deal?

We've seen two reasons why Npgsql has to mess around with your command's SQL: to rewrite your named parameter placeholders into PostgreSQL native positional ones, and to break up any multiple statements for batching. But why should we care about all that?

1. Parsing the PostgreSQL SQL dialect isn't trivial. For example, we must avoid manipulating string literals, which may contain semicolons or text that looks like placeholders. Of course, Npgsql doesn't include a full SQL parser - that would be very hard to do - but rather a very small parser that knows the absolute minimum in order to perform its job. Now, we haven't had any bugs recently, but I'm sure that if I really dove in there, I could produce cases where the parser mistakenly identifies a placeholder or semicolon where it shouldn't, or vice versa. It's an inherently unsafe situation.
2. Beyond correctness, both parsing and producing the rewritten SQL is work, which can hurt performance. The longer the SQL query and the more parameters it has, the more overhead this process adds to query execution. Nobody wants that.
3. When managing a parameter collection (e.g. `NpgsqlParameterCollection`), we have to maintain an internal dictionary that indexes parameters by their name. If we didn't have to handle names, the collection would become a simple list - this is more efficient.
4. Lastly and most importantly, I hate it. I believe a database driver's job is to transmit the SQL users give it, without manipulating it in any way. Simple, easy, efficient, no frills.

So what can be done about this?

## Going raw

The first step towards removing SQL manipulation is the introduction of a proper, 1st-class batching API: rather than packing multiple statements into a single semicolon-delimited string, a structured API would allow the user to manage multiple statements within a batch. The driver would then receive a batch which is already broken down into the various statements, and would no longer need to search for semicolons. The upcoming .NET 6.0 features [a new ADO.NET batching API](https://github.com/dotnet/runtime/issues/28633) which does precisely this:

```c#
var batch = new NpgsqlBatch(conn)
{
    BatchCommands =
    {
        new("SELECT * FROM employees"),
        new("SELECT * FROM departments"),
    }
};
```

The question of parameter placeholders is a bit trickier. Starting with Npgsql 6.0, you can do the following:

```c#
var cmd = new NpgsqlCommand("SELECT * FROM employees WHERE first_name = $1 AND age = $2", conn)
{
    Parameters =
    {
        new() { Value = "Shay" },
        new() { Value = 18 }
    }
};
```

Our parameters no longer have names! And since that's the case ([`NpgsqlParameter.ParameterName`](https://docs.microsoft.com/dotnet/api/system.data.common.dbparameter.parametername) is null), Npgsql implicitly switches into "raw mode", where it no longer performs any parsing or rewriting of your SQL. One consequence of this is that "legacy batching" - multiple semicolon-separated statements - is no longer supported; if you use positional parameters, you must also use the new batching API. If you use named parameters, Npgsql will continue behaving as before, rewriting your SQL in order to maintain full backwards compatibility.

Everything seems to be neatly taken care of - except for one small point. If your command has no parameters at all, Npgsql cannot be sure that there isn't a semicolon hiding somewhere in your SQL, and must grudgingly fall back to parsing; not doing so would break backwards compatibility. So we added an AppContext switch which allows opting into raw mode everywhere, always:

```c#
AppContext.SetSwitch("Npgsql.EnableSqlRewriting", false);

var cmd = new NpgsqlCommand("SELECT * FROM employees", conn);
```

Disabling rewriting will make your queries fail if they contain any named parameters or make use of semicolons for batching. Aside from optimizing the zero-parameters case, this switch can ensure that your application is always communicating in the safest and most efficient way with PostgreSQL.

## Epilog

All the above is available in Npgsql as of 6.0.0-preview7. Unfortunately, positional parameters and the new batching API require changes in layers used over Npgsql: I'm not sure to what extent Dapper supports positional parameters, and EF Core requires some changes in order to support everything too; it's unfortunately too late in the EF Core release cycle to make that happen, but I plan to work on that for EF Core 7.0.

One last point... The new .NET batching API wasn't introduced just so that Npgsql could avoid parsing its SQL. While SQL Server does natively support multiple semicolon-separated statements in a single command (or "batch" in SQL Server parlance), there are some significant drawbacks for doing so - [read this old post for the details](https://docs.microsoft.com/en-us/archive/blogs/dataaccess/does-ado-net-update-batching-really-do-something). We also have good reason to believe that the MySQL provider can benefit from a better batching API as well - so  lots to look forward to.

Oh, and thanks to [@NinoFloris](https://github.com/NinoFloris/) for some very helpful conversations on this!

**UPDATE 2022-05-09**: Amazing timing... PostgreSQL 14 has introduced new syntax which breaks Npgsql's SQL parsing logic, and will probably be non-trivial to recognize properly... see [this issue](https://github.com/npgsql/npgsql/issues/4445). This shows what a bad idea it is for a driver to be parsing SQL.