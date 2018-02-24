---
title: Support for new PostgreSQL 10 features
permalink: /postgresql-10-features
modified: 2018-02-24
comments: true
---
PostgreSQL 10 added [a ton of exciting new features](https://www.postgresql.org/docs/devel/static/release-10.html) which everyone should look at. Npgsql and the EF Core provider have been updated to support those features which require it.

# SCRAM-SHA-256

PostgreSQL 10 adds a new authentication option - SCRAM-SHA-256 - which is superior to MD5 and should be preferred. It will also be possible to add other SASL/SCRAM mechanisms in the future.

Support for SCRAM-SHA-256 has been added and will be released with Npgsql 3.2.7 ([#1530](https://github.com/npgsql/npgsql/issues/1530)).

# Identity columns

Auto-increment in databases is always annoying non-portable. PostgreSQL has had its `serial` pseudo-type, which in reality is just shorthand for a regular `bigint` column with a default value coming from a sequence. While this works fine, it creates some management issues which can get tricky as you evolve your database schema. PostgreSQL 10 introduced standards-compliant identity columns which address these issues, and should now be the recommended way to do autoincrement in PostgreSQL - don't use `serial` anymore for new databases. For more information on `serial` vs. identity and why the latter is better, [see this blog post](https://blog.2ndquadrant.com/postgresql-10-identity-columns/).

At the ADO.NET level Npgsql doesn't really need to know about identity columns - a small fix has been applied to properly detect them when return resultset metadata ([#1789](https://github.com/npgsql/npgsql/issues/1789)).

However, at the EF Core level things get much more interesting for version 2.1. For backwards compatibility reasons, `serial` columns are still the default for columns that are value-generated on add (e.g. int keys). However, you now have the option to specify "identity by default" or "identity always" on properties to override this ("identity always" means that the application cannot provide a value). This can be done on a property-by-property basis, or simply once on your entire model:

```c#
protected override void OnModelCreating(ModelBuilder builder) {
    builder.ForNpgsqlUseIdentityColumns();
}
```

The EF Core provider will even *migrate* your existing `serial` columns to the new identity columns, if you add the above to an existing model. The current serial values will transferred across and new rows won't conflict with existing ones. Needless to say, back up your database before attempting this.

# Sequence data types

This one is definitely minor. PostgreSQL sequences used to be limited to the `bigint` datatype, but version 10 allows you to specify `integer` and `smallint` as well. The EF Core provider has been updated to correctly manage this ([#301](https://github.com/npgsql/Npgsql.EntityFrameworkCore.PostgreSQL/issues/301)).

# macaddr8

PostgreSQL added a new `macaddr8` datatype which allows storing 8-byte MAC address. Support has been added and will be released with Npgsql 3.2.7 ([#1806](https://github.com/npgsql/npgsql/issues/1806)).
