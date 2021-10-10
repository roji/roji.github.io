---
title: Mapping .NET Timestamps to PostgreSQL
permalink: /mapping-timestamps
modified: 2021-10-10
comments: true
---
Npgsql 6.0 contains some significant changes to how timestamps are mapped between .NET and PostgreSQL; while these

## PostgreSQL timestamps

PostgreSQL has

* Naming of `timestamp with time zone` is awful, especially given what `time with time zone` means (but maybe this is standard-mandated).
* Implicit conversions between timestamp and timestamptz are a bad pit of failure.
* timetz is bad (but standard-mandated). Contains an offset rather than a time zone.
* timestamp without time zone is sometimes a local timestamp (e.g. when it gets converted to timestamptz), and sometimes it's "unspecified".
