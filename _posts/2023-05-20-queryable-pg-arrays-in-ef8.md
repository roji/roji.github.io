---
title: Queryable PostgreSQL arrays in EF Core 8.0
permalink: /queryable-pg-arrays-in-ef8
modified: 2023-05-20
comments: true
---
## Queryable collections?

EF Core 8.0 preview4 has just been released, and one of the big features it introduces is queryable primitive collections. This is a really cool feature that allows mapping primitive collections (e.g. `int[]`) to the database, and performing all imaginable LINQ queries over them. Before reading further here, please read the [EF Core blog post](https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4) on this feature; more info and examples are also available in the [EF What's New documentation](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/whatsnew#collections-of-primitive-types).

The rest of this post will discuss PostgreSQL-specific aspects of this feature, which is also fully supported starting with version 8.0.0-preview.4 of the PostgreSQL EF provider.

## Contains over parameter

The EF blog post starts with [a tricky problem](https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/#translating-linq-contains-with-a-parameter-collection): how to translate the LINQ Contains operator when the list of values is a parameter?

```c#
var names = new[] { "Blog1", "Blog2" };

var blogs = await context.Blogs
    .Where(b => names.Contains(b.Name))
    .ToArrayAsync();
```

The solution introduced in preview4 serializes the `names` .NET array into a string containing a JSON array representation, and then uses a SQL function to parse the values out in SQL. Here's the SQL Server sample:

```sql
Executed DbCommand (49ms) [Parameters=[@__names_0='["Blog1","Blog2"]' (Size = 4000)], CommandType='Text', CommandTimeout='30']

SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE EXISTS (
    SELECT 1
    FROM OpenJson(@__names_0) AS [n]
    WHERE [n].[value] = [b].[Name])
```

The nice thing about PostgreSQL, is that it has full, first-class support for array types in the database - this is quite a unique feature. So we don't have to mess around with JSON at all - we can simply send the .NET array directly as a parameter and use it in SQL as follows:

```sql
Executed DbCommand (10ms) [Parameters=[@__names_0={ 'Blog1', 'Blog2' } (DbType = Object)], CommandType='Text', CommandTimeout='30']

SELECT b."Id", b."Name"
FROM "Blogs" AS b
WHERE b."Name" = ANY (@__names_0)
```

In fact, the EF PostgreSQL provider has done this for a few years already, freeing PostgreSQL users with the performance problems that users of other databases had to contend with ([see this issue](https://github.com/dotnet/efcore/issues/13617)). So preview4 doesn't bring any improvements around this specific problem - we were already doing the optimal thing.

## Fully queryable arrays

However, even though the EF PostgreSQL array has supported arrays, its support for querying over them has been quite limited. Now, EF 8.0 preview4 unlocks generalized LINQ querying over primitive collections - once again by converting them to JSON, and using a SQL function to unpack them to a relational rowset. For example, the following LINQ query:

```c#
var tags = new[] { "Tag1", "Tag2" };

var blogs = await context.Blogs
    .Where(b => b.Tags.Intersect(tags).Count() >= 2)
    .ToArrayAsync();
```

... is now translated to the following SQL with SQL Server:

```sql
Executed DbCommand (48ms) [Parameters=[@__tags_0='["Tag1","Tag2"]' (Size = 4000)], CommandType='Text', CommandTimeout='30']

SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE (
    SELECT COUNT(*)
    FROM (
        SELECT [t].[value]
        FROM OPENJSON([b].[Tags]) AS [t] -- column collection
        INTERSECT
        SELECT [t1].[value]
        FROM OPENJSON(@__tags_0) AS [t1] -- parameter collection
    ) AS [t0]) >= 2
```

This uses the SQL Server `OpenJson` function to unpack the JSON array column and parameter into rowsets, over which the LINQ operators are translated in the standard way.

Now let's see how this works on PostgreSQL!

First, here's our .NET entity type:

```c#
public class Blog
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public string[] Tags { get; set; }
}
```

This creates the following table:

```sql
CREATE TABLE "Blogs" (
  "Id" integer GENERATED BY DEFAULT AS IDENTITY,
  "Name" text,
  "Tags" text[] NOT NULL,
  CONSTRAINT "PK_Blogs" PRIMARY KEY ("Id")
);
```

Note that `Tags` is a PostgreSQL array - `text[]`, and not a simple string column containing a JSON array. Aside from mapping .NET arrays more directly and naturally, this has the following advantages:

* It's stored more efficiently: array elements are stored in the same efficient binary encoding that PostgreSQL uses for regular, non-arrays values.
* It's also transferred more efficiently. The same binary encoding is used when reading and writing the elements, meaning that we don't need to constantly serialize and parse JSON.
* Arrays provide more database type safety; it's impossible for the column to contain anything than the defined array type. Similar type safety may be achievable with a JSON array via complex check constraints, but this is more complicated and probably less efficient.

The LINQ query above translates to the following SQL:

```sql
Executed DbCommand (14ms) [Parameters=[@__tags_0={ 'Tag1', 'Tag2' } (DbType = Object)], CommandType='Text', CommandTimeout='30']

SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE (
  SELECT count(*)::int
  FROM (
      SELECT t.value
      FROM unnest(b."Tags") AS t(value)
      INTERSECT
      SELECT t1.value
      FROM unnest(@__tags_0) AS t1(value)
  ) AS t0) >= 2
```

Where SQL Server had the `OPENJSON` function, on PostgreSQL we use the [`unnest`](https://www.postgresql.org/docs/current/functions-array.html#id-1.5.8.25.6.2.2.17.1.1.1) function to expand the array into a relational rowset; conceptually things are very similar, except that the thing being expanded is a native PostgreSQL array rather than a string value containing a JSON array.

So far, so good: we can use arbitrary LINQ operators to query PostgreSQL array columns (and parameters), and the EF provider translates those by "unnesting" the array and then using regular SQL over that.

## And a bonus: PostgreSQL specialized translations

We could stop there - queryable arrays are already a powerful, flexible new mechanism for your LINQ queries. But PostgreSQL also provides a rich set of functions and operators for working with arrays - far beyond what's possible with JSON arrays in other databases. For example, let's say you want to index an element in the array:

```c#
var blogs = await context.Blogs
    .Where(b => b.Tags[2] == "foo")
    .ToArrayAsync();
```

On SQL Server this translates to the following:

```sql
SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE CAST(JSON_VALUE([p].[Ints], '$[1]') AS int) = 10
```

... whereas on PostgreSQL, we can simply do the following:

```sql
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE b."Tags"[3] = 'foo'
```

For something more complex, the provider can translate queries of the following form:

```c#
var tags = new[] { "Tag1", "Tag2" };

var blogs = await context.Blogs
    .Where(b => tags.All(t => b.Tags.Contains(t)))
    .ToArrayAsync();
```

This, in effect, queries Blogs where the Tags column contains all elements in the `tags` parameter. It so happens that PostgreSQL has an array containment operator, so we translate this to:

```sql
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE @__tags_0 <@ b."Tags"
```

This translation - and several like it - have been implemented for several years already; but 8.0.0-preview.4 brings a few new ones. For example:

```c#
var tags = new[] { "Tag1", "Tag2" };

var blogs = await context.Blogs
    .Where(b => b.Tags.Intersect(tags).Any())
    .ToArrayAsync();
```

This queries Blogs where there's any overlap between the Tags column and the `tags` parameter:

```sql
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE b."Tags" && @__tags_0
```

Moving on from set operations, we now also translate Skip and Take to array slicing operations. For example:

```c#
var blogs = await context.Blogs
    .Where(b => b.Tags.Skip(2).Contains("Tag1"))
    .ToArrayAsync();
```

... translates to:

```sql
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE 'Tag1' = ANY (b."Tags"[3:])
```

Note that the C# 2 has been transformed to a 3, since PostgreSQL arrays are 1-based, not 0-based. We can do the same for Take:

```c#
var blogs = await context.Blogs
    .Where(b => b.Tags.Take(2).Contains("Tag1"))
    .ToArrayAsync();
```

... which translates to:

```sql
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE 'Tag1' = ANY (b."Tags"[:2])
```

... and even combines the two, generating:

```c#
SELECT b."Id", b."Name", b."Tags"
FROM "Blogs" AS b
WHERE 'Tag1' = ANY (b."Tags"[2:3])
```

For all the specialized translations supported by the provider, [see this doc page](https://www.npgsql.org/efcore/mapping/array.html). But remember - even if a specialized translation isn't available, the provider will now use `unnest` to expand your array to a rowset, and then employ standard SQL to compose query operators on top of it.

### Summary

The PostgreSQL provider has supported arrays for quite a while, but 8.0.0-preview.4 brings a major upgrade to array support: arbitrary LINQ operators can now be used, and some specialized PostgreSQL translations have been added to make your SQL tighter and more efficient. Let us know about cool querying ideas or any bugs!