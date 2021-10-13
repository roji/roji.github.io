---
title: Mapping .NET Timestamps to PostgreSQL
permalink: /postgresql-dotnet-timestamp-mapping
modified: 2021-10-10
comments: true
---
Npgsql 6.0 contains some significant changes to how timestamps are mapped between .NET and PostgreSQL - most applications will need to react to this (although a compatibility flag exists). This post gives the context for these changes, going over the timestamp types on both sides and the problems in mapping them.

## PostgreSQL timestamps

Like with most things, PostgreSQL conforms to the SQL standard when it comes to timestamps ([full docs](https://www.postgresql.org/docs/current/datatype-datetime.html)): it has a `timestamp without time zone` and a `timestamp with time zone` type (the shorter aliases are `timestamp` and `timestamptz`). `timestamptz` is perhaps the worst-named type in the world: it does **not**, in fact, store a time zone in the database, but rather a UTC timestamp; that causes lots of confusion from users expecting to persist a full timezone-aware timestamp to PostgreSQL. In this sense, `timestamptz` is different from the SQL Server [`datetimeoffset`](https://docs.microsoft.com/sql/t-sql/data-types/datetimeoffset-transact-sql) type (but see note below on why offsets may be a bad idea). What `timestamptz` **is** good for, is storing and interacting with UTC timestamps, or globally agreed-upon points in time, where the time zone does not matter. For example, when recording the time a transaction took place, you typically store a UTC timestamp, and then display it in the user's local time zone, as reported by their web browser; this allows you to show the same timestamp to multiple users, each in their own time zone, and also to support the fact that users may be in one time zone today, and in another tomorrow. This is sometimes called doing "UTC everywhere", and it tends to work well as a default pattern. In the relatively rarer cases where you need to store the time zone along with a timestamp, a separate column must be used alongside your timestamp column, typically holding a string representation of the timezone (e.g. `Europe/Berlin`)[^1].

The other type - `timestamp` - can be used to store a timestamp whose time zone is unknown, implicit or assumed to be local. It's really important to understand that this does not represent a specific point in time unless coupled with some time zone: the same date/time combination corresponds to different universal instances in different time zones. PostgreSQL does have a [`TimeZone`](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-TIMEZONE) connection state parameter, which defines the "local time zone" of the connection; it's defined in your PostgreSQL configuration by default, and can be changed in your connection. When converting a `timestamp` into a `timestamptz` (remember: the latter means "UTC"), PostgreSQL will treat your `timestamp` as a local timestamp, and convert it to UTC based on the connection's current `TimeZone`. However, fiddling around with your connection's `TimeZone` and depending on your database to do timezone conversions usually isn't a practical way to do things - you typically want to store and retrieve UTC timestamps from your database, and do any conversions to/from local timezones in your application, when interacting with users.

## .NET timestamps

The .NET situation around timestamps is... not pretty... .NET has some basic flaws in this area which have been with us since the beginning of time, and cannot be corrected without introducing unacceptable breaking changes. The .NET timestamp arsenal includes two main types: [`DateTime`](https://docs.microsoft.com/dotnet/api/system.datetime) and [`DateTimeOffset`](https://docs.microsoft.com/dotnet/api/system.datetimeoffset).

`DateTime` unsurprisingly contains a date and a time, but also a [Kind](https://docs.microsoft.com/dotnet/api/system.datetimekind) property which can be `Utc`, `Local` or `Unspecified`: `Utc` is pretty self-explanatory, `Local` means a timestamp in the timezone of the machine where .NET is running, and `Unspecified` is, well, not very specified. One problematic aspect of DateTime is that these very different concepts are represented via the same .NET type: if a function accepts a DateTime, which Kind should you pass in? What happens when you compare a UTC DateTime with an Unspecified one? (The answer is that the timestamps will be compared disregarding the Kind, which I can't imagine can produce fruitful results in any sane application). To know more about DateTime's failings, see this [excellent blog post](https://blog.nodatime.org/2011/08/what-wrong-with-datetime-anyway.html) by Jon Skeet.

DateTimeOffset is at least less ambiguous than DateTime: it's a date and time, plus a timezone offset. Taken together, these identify a specific instant in time, and so a DateTimeOffset can always be unambiguously converted to a UTC timestamp, if needed. Its API still has some issues (see Jon's post above), but in my opinion, the main problem with this type is that it gives the illusion of being timezone-aware without delivering on it. An offset (e.g. `UTC+01:00`) is **not** a timezone (e.g. IANA/Olson `Europe/Berlin`): timezones contain information about daylight saving time, which a simple offset does not; Berlin is sometimes at `UTC+01:00` and sometimes at `UTC+02:00`. This is especially important if you're going to do arithmetic on a timestamp: if you add a few hours to a timestamp, an accurate result would have to take daylight savings into account. And if you're not doing arithmetic, then you may not need the timezone in the first place (why not just use UTC?). The same criticism goes for the SQL Server `datetimeoffset` type: it makes you think you're good, while neglecting daylight savings time.

Oh, and since I mentioned Jon Skeet above, you should absolute take a look at his [NodaTime](https://nodatime.org/) library: this is how date/time types are done right. I'd recommend that any serious application that needs to deal with timestamps seriously consider using it, and Npgsql even fully supports it (both at the [ADO.NET](https://www.npgsql.org/doc/types/nodatime.html) and [EF Core](https://www.npgsql.org/efcore/mapping/nodatime.html) levels).

## Mapping .NET to PostgreSQL

One of the tasks of a database driver is to map two different type systems to one another; in our case, the .NET types must be mapped to the PostgreSQL ones. The mapping is sometimes simple (e.g. .NET `long` corresponds perfectly to a PostgreSQL `bigint`), but sometimes it's quite complex. You guessed it: timestamps fall in the latter basket.

One curious thing with PostgreSQL `timestamptz`, is that while it's stored as a UTC timestamp in the database, its textual representation is a local timestamp based on the `TimeZone` connection parameter: reading a `timestamptz` as text yields something like `2004-10-19 10:23:54+02`. Unfortunately this odd behavior shaped Npgsql's original timestamp mapping in a significant way: reading a `timestamptz` returns a Local DateTime[^1]. Among other things, this means you cannot round-trip a UTC DateTime: you can send it just fine, but when you read it back, you get a converted local timestamp. A similar thing was done with DateTimeOffset: Npgsql converted it to UTC before sending, and returned a DateTimeOffset in the machine's time zone when reading (remember, no timezone or offset is actually stored in the database!): if I send a DateTimeOffset with offset `+02:00` on a machine configured with offset `+01:00`, it would be saved to UTC but read back with `+01:00`, with Npgsql doing all the conversions. This state of affairs led to a lot of general confusion, and made it quite difficult to support simple "UTC everywhere" programming, where you send a UTC timestamp to the database, and read it back in the same way. 

In Npgsql 6.0, we redid the timestamp mapping with the following principles in mind:

* 1st-class support for the "UTC everywhere" pattern, and promote it as the default timestamp strategy.
* Cleanly separate between UTC timestamps and non-UTC timestamps as two different types, and disallow mixing them to protect against accidental errors.
* Values should always be round-trippable - whatever you send to PostgreSQL, you should get the same thing back. If we can't roundtrip it, we should refuse to write it.
* Values should never undergo any implicit timezone conversions when being read or written. Any conversions should be done by the user, making them clear and explicit in the code.

This means the following concrete things:

* We now send UTC DateTime as `timestamptz`, and Local/Unspecified DateTime as `timestamp`; trying to send a non-UTC DateTime as `timestamptz` will throw an exception, etc. In effect, Npgsql is creating a strict type distinction between the different DateTime Kinds (which is how they should have been represented in the first place in .NET).[^2]
* We only allow sending a DateTimeOffset with offset 0 (as `timestamptz`): since the offset isn't stored in the database, it can't be round-tripped.[^3]
* Reading a `timestamptz` will now yield a UTC DateTime or DateTimeOffset - no more implicit conversions.
* By nature, EF Core must make mapping decisions solely based on the type (DateTime) and cannot take the Kind into account, so we have to pick one type or the other as the default. Since "UTC everywhere" should generally be preferred, we now map DateTime to `timestamptz`.[^4]
* Corresponding changes were done to the NodaTime mappings, though the situation is much simpler there, since the different concepts are represented by different .NET types (e.g. [Instant vs. LocalDateTime](https://nodatime.org/3.0.x/userguide/concepts)).

As you can imagine, the above implies a lot of breaking changes... This is not something we did lightly, but we do believe our users will end up in a better place. However, we've also provided a backwards compatibility flag which allows reverting to the previous behavior; [see the documentation](https://www.npgsql.org/doc/types/datetime.html).

Please let us know what you think! Don't hesitate to open questions on the [Npgsql](https://github.com/npgsql/npgsql) or [EF Core provider](https://github.com/npgsql/efcore.pg) repos, or to ping me [on twitter]().

## Appendix: what PostgreSQL (or the SQL standard) got wrong

For those interested, here are a few thoughts on flaws in the PostgreSQL timestamp system - though the SQL standard is probably the one at fault here. There's not much to be done about these, but it's important to be aware of them.

* The naming is quite atrocious:
    * `timestamp with time zone` has a timezone in its textual representation, but not in its storage.
    * This is also inconsistent with the type `time with time zone`, which *does* store an offset in the database.
    * The `timestamp` name is bound to make people use it as the default, though it probably is not the thing most applications want.
* PostgreSQL implicitly casts between `timestamp` and `timestamptz`, making it easy to accidentally get a timezone conversion; it would have been better to require explicit conversions instead. For example, the [`extract`](https://www.postgresql.org/docs/13/functions-datetime.html#FUNCTIONS-DATETIME-TABLE) function accepts a `timestamp`, so passing a `timestamptz` would cause an implicit timezone conversion.
* It's arguably a bad idea for the `timestamptz` textual representation to contain a local representation.
* `timestamp` is sometimes treated as a local timestamp (e.g. when converting it to `timestamptz`), but sometimes is simply an unspecified timestamp.
* `time with time zone` makes no sense.

[^1]:
    For more information on when "UTC Everywhere" is less appropriate and how to deal with it, see this great [post](https://codeblog.jonskeet.uk/2019/03/27/storing-utc-is-not-a-silver-bullet/) by Jon Skeet.

[^2]:
    Note that Npgsql did the timezone conversion based on the machine's timezone, rather than based on the PostgreSQL `TimeZone`, so did not match the PostgreSQL behavior in any case. This was because .NET had no way of parsing the PostgreSQL IANA/Olson timezone IDs until [.NET 6.0](https://devblogs.microsoft.com/dotnet/date-time-and-time-zone-enhancements-in-net-6/).

[^3]:
    Incidentally, this is the only case in the Npgsql type mapping system where the PostgreSQL type depends not only on the CLR type (DateTime), but also on its value (the Kind). This isn't trivial to do, and especially to do efficiently - thanks once again, DateTime!

[^4]:
    There are a few specific cases where we allow non-round-trippability. For one, PostgreSQL has only microsecond precision, whereas the .NET types have tick precision (100 nanoseconds); the driver silently truncates the extra precision rather than throwing. We also allow writing `default(DateTime)` as `timestamptz` even though its Kind is Unspecified.

[^5]:
    This means that version 6.0 of the EF Core provider triggers creation of a migration of all DateTime properties from `timestamp` to `timestamptz` - this should be done with care. [The release notes](https://www.npgsql.org/efcore/release-notes/6.0.html#migrating-columns-from-timestamp-to-timestamptz) provide advice on how to safely do this.
