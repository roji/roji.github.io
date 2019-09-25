---
title: EFCore 3.0 for PostgreSQL - Advanced JSON Support
permalink: /efcore-pg-advanced-json
modified: 2019-09-26
comments: true
---

# JSON and Databases

Most relational databases have had some sort of native support for JSON for quite a while now; PostgreSQL introduced its first JSON support in version 9.2, back in 2012, and the more optimized `jsonb` type in 2014. JSON types have in part been the relational database response to the NoSQL movement, with its pervasive, schema-less JSON documents: look, we can do it too! But the marriage of a traditional relational schema with non-relational documents has proven very powerful indeed; complex data no longer have to be represented via sprawling, relational models involving endless joins, and islands of fluid, schema-less content within a stricter relational model brought some very welcome flexibility.

Database JSON support usually means that some operations on JSON data can be performed in the database; after all, simply storing and loading JSON documents is quite useless in itself. For JSON to really shine, we need to be able to ask for all JSON documents satisfying some condition - and to do it efficiently. At the most basic level, PostgreSQL supports the following:

```sql
SELECT * FROM some_table WHERE customer->>'name' == 'Joe';
```

Assuming the `some_table` table has a JSON column named `customer`, this query will make PostgreSQL examine each row's document, and return those rows where the `name` key is equal to "Joe". [Proper indexing](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING) can make this perform very fast, and PostgreSQL has [a plethora of other operators and functions](https://www.postgresql.org/docs/current/functions-json.html) that can be used to construct JSON queries.

Now, the syntax above is entirely PostgreSQL-specific: other databases have others ways to express queries. SQL/JSON standardization is underway, and PostgreSQL 12 [will support jsonpath queries](https://paquier.xyz/postgresql-2/postgres-12-jsonpath/) which should finally provide a cross-database way to describe JSON queries. Unfortunately, the non-standardized nature of JSON support has meant that ORMs have often stayed away from it, and developers have been forced to drop down to raw SQL if they wanted to access JSON goodness.

No more! Release 3.0.0 of the Npgsql Entity Framework Core provider for PostgreSQL brings some exciting new JSON support, leveraging a unique feature of C#'s LINQ to express database JSON queries in a strongly-typed and natural way. The rest of this post will present the key new features, [consult the documentation for a more complete description](http://www.npgsql.org/efcore/mapping/json.html).

# Strongly-typed access via POCOs

Without further ado, you can now define an EF Core entity as follows:

```c#
public class SomeEntity   // Maps to a database table
{
    public int Id { get; set; }
    [Column(TypeName = "jsonb")]
    public Customer Customer { get; set; }
}

public class Customer    // Maps to a JSON column in the table
{
    public string Name { get; set; }
    public int Age { get; set; }
    public Order[] Orders { get; set; }
}

public class Order       // Part of the JSON column
{
    public decimal Price { get; set; }
    public string ShippingAddress { get; set; }
}
```

Our `SomeEntity` type - which maps to a database table - contains an arbitrary user type (or POCO, plain-old-CLR-object), which is mapped to a PostgreSQL `jsonb` column via the `[Column]` data annotation attribute. This is really *all* you have to do, and everything will work as expected: Npgsql will use the new [System.Text.Json](https://devblogs.microsoft.com/dotnet/try-the-new-system-text-json-apis/) to serialize and deserialize your instances to JSON data. Note also that our POCO, `Customer`, contains an array of another POCO, `Order`; this will also just work as expected, with the array of orders appearing inside the customer's JSON document.

That's it, couldn't be simpler. No need for additional `customer` and `order` tables, with joins all around. But what about querying, as promised above? No problem:

```c#
var joes = context.CustomerEntries
    .Where(e => e.Customer.Name == "Joe")
    .ToList();
```

This will produce the PostgreSQL-specific JSON syntax we saw above. Once again: we're using natural C# and LINQ to express an SQL query over a JSON column in our database.

# Weakly-typed access via JsonDocument

Mapping POCOs is great when your JSON documents have a stable schema, but JSON is frequently used precisely when things are fluid: a document in one row could have a certain key which another document might not. A strongly-typed POCO is inappropriate for mapping in these circumstances, but never fear - there's a solution for that as well. System.Text.Json also comes with a Document Object Model (DOM) for accessing JSON documents: you use types such as [`JsonDocument`](https://docs.microsoft.com/en-us/dotnet/api/system.text.json.jsondocument) and [`JsonElement`](https://docs.microsoft.com/en-us/dotnet/api/system.text.json.jsonelement) for weakly-typed access. These can also be mapped:

```c#
public class SomeEntity
{
    public int Id { get; set; }
    public JsonDocument Customer { get; set; }
}

var joes = context.CustomerEntries
    .Where(e => e.Customer.GetProperty("Name").GetString() == "Joe")
    .ToList();
```

This will produce the same SQL as above.

# Closing Words

This hopefully gave a good overview of this new JSON feature, which should make PostgreSQL JSON operations accessible to EF Core users - [the full documentation is available here](http://www.npgsql.org/efcore/mapping/json.html). Based on feedback, the plan is also to look into supporting JSON in other database providers, such as SQL Server or Sqlite; standardized SQL/JSON may provide an opportunity for generic, cross-database support.

Please send positive and negative feedback via twitter ([@shayrojansky](https://twitter.com/shayrojansky)) or by opening an issue [on the provider repo](https://github.com/npgsql/Npgsql.EntityFrameworkCore.PostgreSQL/). And have fun!
