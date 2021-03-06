---
title: Error Messages
menu:
  influxdb_1_0:
    weight: 30
    parent: troubleshooting
---

This page documents errors, their descriptions, and, where applicable,
common resolutions.

<dt>
**Disclaimer:** This document does not contain an exhaustive list of all possible InfluxDB errors.
</dt>

## error parsing query: found < >, expected identifier at line < >, char < >

### InfluxQL Syntax
The `expected identifier` error occurs when InfluxDB anticipates an identifier
in a query but doesn't find it.
Identifiers are tokens that refer to continuous query names, database names,
field keys, measurement names, retention policy names, subscription names,
tag keys, and user names.
The error is often a gentle reminder to double-check your query's syntax.

**Examples**

*Query 1:*
```
> CREATE CONTINUOUS QUERY ON "telegraf" BEGIN SELECT mean("usage_idle") INTO "average_cpu" FROM "cpu" GROUP BY time(1h),"cpu" END
ERR: error parsing query: found ON, expected identifier at line 1, char 25
```
Query 1 is missing a continuous query name between `CREATE CONTINUOUS QUERY` and
`ON`.

*Query 2:*
```
> SELECT * FROM WHERE "blue" = true
ERR: error parsing query: found WHERE, expected identifier at line 1, char 15
```
Query 2 is missing a measurement name between `FROM` and `WHERE`.

### InfluxQL Keywords
In some cases the `expected identifier` error occurs when one of the
[identifiers](/influxdb/v1.0/concepts/glossary/#identifier) in the query is an
[InfluxQL Keyword](/influxdb/v1.0/query_language/spec/#keywords).
To successfully query an identifier that's also a keyword, enclose that
identifier in double quotes.

**Examples**

*Query 1:*
```
> SELECT duration FROM runs
ERR: error parsing query: found DURATION, expected identifier, string, number, bool at line 1, char 8
```
In Query 1, the field key `duration` is an InfluxQL Keyword.
Double quote `duration` to avoid the error:
```
> SELECT "duration" FROM runs
```

*Query 2:*
```
> CREATE RETENTION POLICY limit ON telegraf DURATION 1d REPLICATION 1
ERR: error parsing query: found LIMIT, expected identifier at line 1, char 25
```
In Query 2, the retention policy name `limit` is an InfluxQL Keyword.
Double quote `limit` to avoid the error:
```
> CREATE RETENTION POLICY "limit" ON telegraf DURATION 1d REPLICATION 1
```

While using double quotes is an acceptable workaround, we recommend that you avoid using InfluxQL keywords as identifiers for simplicity's sake.

**Resources:**
[InfluxQL Keywords](/influxdb/v1.0/query_language/spec/#keywords),
[Query Language Documentation](/influxdb/v1.0/query_language/)

## error parsing query: found < >, expected string at line < >, char < >

The `expected string` error occurs when InfluxDB anticipates a string
but doesn't find it.
In most cases, the error is a result of forgetting to quote the password
string in the `CREATE USER` statement.

**Example**

```
> CREATE USER penelope WITH PASSWORD timeseries4dayz
ERR: error parsing query: found timeseries4dayz, expected string at line 1, char 36
```

The `CREATE USER` statement requires single quotation marks around the password
string:

```
> CREATE USER penelope WITH PASSWORD 'timeseries4dayz'
```

Note that you should not include the single quotes when authenticating requests.

**Resources:**
[Authentication and Authorization](/influxdb/v1.0/query_language/authentication_and_authorization/)

## error parsing query: mixing aggregate and non-aggregate queries is not supported

The `mixing aggregate and non-aggregate` error occurs when a `SELECT` statement
includes both an [aggregate function](/influxdb/v1.0/query_language/functions/)
and a standalone [field key](/influxdb/v1.0/concepts/glossary/#field-key) or
[tag key](/influxdb/v1.0/concepts/glossary/#tag-key).

Aggregate functions return a single calculated value and there is no obvious
single value to return for any unaggregated fields or tags.

**Example**

*Raw data:*

The `peg` measurement has two fields (`square` and `round`) and one tag
(`force`):
```
name: peg
---------
time                   square   round   force
2016-10-07T18:50:00Z   2        8       1
2016-10-07T18:50:10Z   4        12      2
2016-10-07T18:50:20Z   6        14      4
2016-10-07T18:50:30Z   7        15      3
```

*Query 1:*

```
> SELECT mean("square"),"round" FROM "peg"
ERR: error parsing query: mixing aggregate and non-aggregate queries is not supported
```
Query 1 includes an aggregate function and a standalone field.

`mean("square")` returns a single aggregated value calculated from the four values
of `square` in the `peg` measurement, and there is no obvious single field value
to return from the four unaggregated values of the `round` field.

*Query 2:*

```
> SELECT mean("square"),"force" FROM "peg"
ERR: error parsing query: mixing aggregate and non-aggregate queries is not supported
```
Query 2 includes an aggregate function and a standalone tag.

`mean("square")` returns a single aggregated value calculated from the four values
of `square` in the `peg` measurement, and there is no obvious single tag value
to return from the four unaggregated values of the `force` tag.

**Resources:**
[Functions](/influxdb/v1.0/query_language/functions/)

## unable to parse < >: bad timestamp

### Timestamp Syntax
The `bad timestamp` error occurs when the
[line protocol](/influxdb/v1.0/concepts/glossary/#line-protocol) includes a
timestamp in a format other than a UNIX timestamp.

**Example**

```
> INSERT pineapple value=1 '2015-08-18T23:00:00Z'
ERR: {"error":"unable to parse 'pineapple value=1 '2015-08-18T23:00:00Z'': bad timestamp"}
```

The line protocol above uses an [RFC3339](https://www.ietf.org/rfc/rfc3339.txt)
timestamp.
Replace the timestamp with a UNIX timestamp to avoid the error and successfully
write the point to InfluxDB:

```
> INSERT pineapple,fresh=true value=1 1439938800000000000
```

### Line Protocol Syntax
In some cases, the `bad timestamp` error occurs with more general syntax errors
in the line protocol.
Line protocol is whitespace sensitive; misplaced spaces can cause InfluxDB
to assume that a field or tag is an invalid timestamp.

**Example**

*Write 1*
```
> INSERT hens location=2 value=9
ERR: {"error":"unable to parse 'hens location=2 value=9': bad timestamp"}
```
The line protocol in Write 1 separates the `hen` measurement from the `location=2`
tag with a space instead of a comma.
InfluxDB assumes that the `value=9` field is the timestamp and returns an error.

Use a comma instead of a space between the measurement and tag to avoid the error:
```
> INSERT hens,location=2 value=9
```

*Write 2*
```
> INSERT cows,name=daisy milk_prod=3 happy=3
ERR: {"error":"unable to parse 'cows,name=daisy milk_prod=3 happy=3': bad timestamp"}
```
The line protocol in Write 2 separates the `milk_prod=3` field and the
`happy=3` field with a space instead of a comma.
InfluxDB assumes that the `happy=3` field is the timestamp and returns an error.

Use a comma instead of a space between the two fields to avoid the error:
```
> INSERT cows,name=daisy milk_prod=3,happy=3
```

**Resources:**
[Line Protocol Tutorial](/influxdb/v1.0/write_protocols/line_protocol_tutorial/),
[Line Protocol Reference](/influxdb/v1.0/write_protocols/line_protocol_reference/)

## write failed for shard < >: engine: cache maximum memory size exceeded

The `cache maximum memory size exceeded` error occurs when the cached
memory size increases beyond the
[`cache-max-memory-size` setting](/influxdb/v1.0/administration/config/#cache-max-memory-size-524288000)
in the configuration file.

By default, `cache-max-memory-size` is set to 512mb.
This value is fine for most workloads, but is too small for larger write volumes
or for datasets with higher [series cardinality](/influxdb/v1.0/concepts/glossary/#series-cardinality).
If you have lots of RAM you could set it to `0` to disable the cached memory
limit and never get this error.
You can also examine the `memBytes` field in the`cache` measurement in the
[`_internal` database](/influxdb/v1.0/troubleshooting/statistics/#internal-monitoring)
to get a sense of how big the caches are in memory.

**Resources:**
[Database Configuration](/influxdb/v1.0/administration/config/)
