---
title: Data Exploration

menu:
  influxdb_1_0:
    weight: 10
    parent: query_language
---

InfluxQL is an SQL-like query language for interacting with data in InfluxDB.
The following sections cover useful query syntax for exploring your data.

<table style="width:100%">
  <tr>
    <td><b>The Basics:</b></td>
    <td><b>Limit and Sort your Results:</b></td>
    <td><b>General Tips on Query Syntax:</b></td>
  </tr>
  <tr>
    <td><a href="#the-select-statement-and-the-where-clause">The SELECT statement and the WHERE clause</a></td>
    <td><a href="#limit-query-returns-with-limit-and-slimit">Limit query returns with LIMIT and SLIMIT</a></td>
    <td><a href="#multiple-statements-in-queries">Multiple Statements in Queries</a></td>
  </tr>
  <tr>
    <td><a href="#the-group-by-clause">The GROUP BY clause</a></td>
    <td><a href="#sort-query-returns-with-order-by-time-desc">Sort query returns with ORDER BY time DESC</a></td>
    <td><a href="#merge-series-in-queries">Merge Series in Queries</a></td>
  </tr>
  <tr>
    <td><a href="#the-into-clause">The INTO clause</a></td>
    <td><a href="#paginate-query-returns-with-offset-and-soffset">Paginate query returns with OFFSET and SOFFSET</a></td>
    <td><a href="#time-syntax-in-queries">Time Syntax in Queries</a></td>
  </tr>
  <tr>
    <td><a href="#"></a></td>
    <td><a href="#"></a></td>
    <td><a href="#data-types-and-cast-operations-in-queries">Data Types and Cast Operations in Queries</a></td>
  </tr>
  <tr>
    <td><a href="#"></a></td>
    <td><a href="#"></a></td>
    <td><a href="#regular-expressions-in-queries">Regular Expressions in Queries</a></td>
  </tr>
</table>

The examples below query data using [InfluxDB's Command Line Interface (CLI)](/influxdb/v1.0/tools/shell/).
See the [Querying Data](/influxdb/v1.0/guides/querying_data/) guide for how to query data directly using the HTTP API.

### Sample data

If you'd like to follow along with the queries in this document, see [Sample Data](/influxdb/v1.0/query_language/data_download/) for how to download and write the data to InfluxDB.

This document uses publicly available data from the [National Oceanic and Atmospheric Administration's (NOAA) Center for Operational Oceanographic Products and Services](http://tidesandcurrents.noaa.gov/stations.html?type=Water+Levels).
The data include water levels (ft) collected every six seconds at two stations (Santa Monica, CA (ID 9410840) and Coyote Creek, CA (ID 9414575)) over the period from August 18, 2015 through September 18, 2015.

A subsample of the data in the measurement `h2o_feet`:
```
name: h2o_feet
--------------
time			                level description	      location	       water_level
2015-08-18T00:00:00Z	  between 6 and 9 feet	   coyote_creek	   8.12
2015-08-18T00:00:00Z	  below 3 feet		          santa_monica	   2.064
2015-08-18T00:06:00Z	  between 6 and 9 feet	   coyote_creek	   8.005
2015-08-18T00:06:00Z	  below 3 feet		          santa_monica	   2.116
2015-08-18T00:12:00Z	  between 6 and 9 feet	   coyote_creek	   7.887
2015-08-18T00:12:00Z	  below 3 feet		          santa_monica	   2.028
2015-08-18T00:18:00Z	  between 6 and 9 feet	   coyote_creek	   7.762
2015-08-18T00:18:00Z	  below 3 feet		          santa_monica	   2.126
2015-08-18T00:24:00Z	  between 6 and 9 feet	   coyote_creek	   7.635
2015-08-18T00:24:00Z	  below 3 feet		          santa_monica	   2.041
```

The [series](/influxdb/v1.0/concepts/glossary/#series) are made up of the [measurement](/influxdb/v1.0/concepts/glossary/#measurement) `h2o_feet` and the [tag key](/influxdb/v1.0/concepts/glossary/#tag-key) `location` with the [tag values](/influxdb/v1.0/concepts/glossary/#tag-value) `santa_monica` and `coyote_creek`.
There are two [fields](/influxdb/v1.0/concepts/glossary/#field): `water_level` which stores floats and `level description` which stores strings.
All of the data are in the `NOAA_water_database` database.

> **Disclaimer:** The `level description` field isn't part of the original NOAA data - we snuck it in there for the sake of having a field key with a special character and string [field values](/influxdb/v1.0/concepts/glossary/#field-value).

<br>
<br>
# The SELECT statement and the `WHERE` clause
InfluxQL's `SELECT` statement follows the form of an SQL `SELECT` statement where the `WHERE` clause is optional:
```sql
SELECT <stuff> FROM <measurement_name> [WHERE <some_conditions>]
```  

## The basic `SELECT` statement
The following three examples return everything from the measurement `h2o_feet`.
While they all return the same result, they get to that result in slightly different ways and serve to introduce some of the specifics of the `SELECT` syntax.

#### Query **A**: Select all tags and fields
```sql
> SELECT * FROM "h2o_feet"
```  
#### Query **B**: Select specific tags and fields
```sql
> SELECT "level description","location","water_level" FROM "h2o_feet"
```  
#### Query **C**: Select specific tags and fields and their identifier type
```sql
> SELECT "level description"::field,"location"::tag,"water_level"::field FROM "h2o_feet"
```

#### Explanation
Query **A** selects everything from `h2o_feet` with `*`.

Queries **B** and **C** select everything from `h2o_feet` by specifying each tag
key and field key in the measurement.
Things to note:

* Separate multiple fields and tags of interest with a comma.
* You must specify at least one field in the `SELECT` statement.
* While not always necessary, we recommend that you double quote identifiers.
Identifiers **must** be double quoted if they contain characters other than `[A-z,0-9,_]`, or if they are an [InfluxQL keyword](https://github.com/influxdb/influxdb/blob/master/influxql/README.md#keywords).
Identifiers are continuous query names, database names, field keys, measurement names,
retention policy names, subscription names, tag keys, and user names.

Query **C** specifies if the identifier is a field or tag by including the
syntax `<identifier>::<field>` or `<identifier>::<tag>`.
Include this syntax to differentiate between field keys and tag keys that have
the same name.
In addition, you can specify a field's [type](/influxdb/v1.0/write_protocols/write_syntax/#data-types) with the syntax
`<field_key>::<type>`.
We'll go into detail about this functionality in the
[casting section](#data-types-and-cast-operations-in-queries) of this document.

If you're following along with the [sample data](#sample-data), here's the CLI response for all
three queries:
 ```
name: h2o_feet
--------------
time			                level description	      location	       water_level
2015-08-18T00:00:00Z	  between 6 and 9 feet	   coyote_creek	   8.12
2015-08-18T00:00:00Z	  below 3 feet		          santa_monica	   2.064
2015-08-18T00:06:00Z	  between 6 and 9 feet	   coyote_creek	   8.005
2015-08-18T00:06:00Z	  below 3 feet		          santa_monica	   2.116
[...]
2015-09-18T21:24:00Z	  between 3 and 6 feet	   santa_monica	   5.013
2015-09-18T21:30:00Z	  between 3 and 6 feet	   santa_monica	   5.01
2015-09-18T21:36:00Z	  between 3 and 6 feet	   santa_monica	   5.066
2015-09-18T21:42:00Z	  between 3 and 6 feet	   santa_monica	   4.938
```

### `FROM` clause syntax

The `FROM` clause specifies the relevant measurement(s) for your query.
In examples A-C above we queried a single measurement (`h2o_feet`).
This section covers additional syntax options for the `FROM` clause.

#### Query **D**: Specify multiple measurements
```
> SELECT * FROM "h2o_feet","h2o_pH"
```

#### Query **E**: Fully qualify a measurement's database and retention policy
```
> SELECT * FROM "NOAA_water_database"."autogen"."h2o_feet"
```

#### Query **F**: Fully qualify a measurement's database
```
> SELECT * FROM "NOAA_water_database".."h2o_feet"
```

#### Explanation

Query **D** selects everything from two measurements: `h2o_feet` and `h2o_pH`.
Note that we separate multiple measurements with a comma.

Queries **E** and **F** select everything from `h2o_feet` by fully qualifying the
`h2o_feet` measurement.
Fully qualify a measurement if you wish to query data from a different database
or from a retention policy other than the `DEFAULT` retention policy.
Query **E** specifies the measurement's database and
[retention policy](/influxdb/v1.0/concepts/glossary/#retention-policy-rp) with
the syntax:

```
"<database_name>"."<retention_policy_name>"."<measurement_name>"
```

Query **F** specifies the measurement's database with the syntax below.
Note that the `..` informs InfluxDB to query the data in the database's
`DEFAULT` retention policy.

```
"<database_name>".."<measurement_name>"
```

## The `SELECT` statement and arithmetic
Perform basic arithmetic operations on fields that store floats and integers.

Add two to the field `water_level`:
```sql
> SELECT "water_level" + 2 FROM "h2o_feet"
```
CLI response:
```bash
name: h2o_feet
--------------
time
2015-08-18T00:00:00Z	10.12
2015-08-18T00:00:00Z	4.064
[...]
2015-09-18T21:36:00Z	7.066
2015-09-18T21:42:00Z	6.938
```

Another example that works:
```sql
> SELECT ("water_level" * 2) + 4 from "h2o_feet"
```
CLI response:
```bash
name: h2o_feet
--------------
time
2015-08-18T00:00:00Z	20.24
2015-08-18T00:00:00Z	8.128
[...]
2015-09-18T21:36:00Z	14.132
2015-09-18T21:42:00Z	13.876
```

> **Note:** When performing arithmetic on fields that store integers be aware that InfluxDB casts those integers to floats for all mathematical operations.
This can lead to [overflow issues](/influxdb/v1.0/troubleshooting/frequently-asked-questions/#what-are-the-minimum-and-maximum-integers-that-influxdb-can-store) for some numbers.

## The `WHERE` clause
Use a `WHERE` clause to filter your data based on tags, time ranges, and/or field values.

> **Note:** The quoting syntax for queries differs from the [line protocol](/influxdb/v1.0/concepts/glossary/#line-protocol).
Please review the [rules for single and double-quoting](/influxdb/v1.0/troubleshooting/frequently-asked-questions/#when-should-i-single-quote-and-when-should-i-double-quote-in-queries) in queries.

**Tags**  
Return data where the tag key `location` has the tag value `santa_monica`:  
```sql
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica'
```
* Always single quote tag values in queries - they are strings.
Note that double quotes do not work when specifying tag values and can cause queries to silently fail.


> **Note:** Tags are indexed so queries on tag keys or tag values are more performant than queries on fields.

Return data where the tag key `location` has no tag value (more on regular expressions [later](/influxdb/v1.0/query_language/data_exploration/#regular-expressions-in-queries)):
```sql
> SELECT * FROM "h2o_feet" WHERE "location" !~ /./
```

Return data where the tag key `location` has a value:
```sql
> SELECT * FROM "h2o_feet" WHERE "location" =~ /./
```
**Time ranges**  
Return data from the past seven days:
```sql
> SELECT * FROM "h2o_feet" WHERE time > now() - 7d
```
* `now()` is the Unix time of the server at the time the query is executed on that server.
For more on `now()` and other ways to specify time in queries, see [time syntax in queries](/influxdb/v1.0/query_language/data_exploration/#time-syntax-in-queries).

**Field values**  
Return data where the tag key `location` has the tag value `coyote_creek` and the field `water_level` is greater than 8 feet:
```sql
> SELECT * FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND  "water_level" > 8
```
Return data where the tag key `location` has the tag value `santa_monica` and the field `level description` equals `'below 3 feet'`:
```sql
> SELECT * FROM "h2o_feet" WHERE "location" = 'santa_monica' AND "level description" = 'below 3 feet'
```
Return data where the field values in `water_level` plus `2` are greater than `11.9`:
```sql
> SELECT * FROM "h2o_feet" WHERE "water_level" + 2 > 11.9
```

* Always single quote field values that are strings.
Note that double quotes do not work when specifying string field values and can cause queries to silently fail.

> **Note:** Fields are not indexed; queries on fields are not as performant as those on tags.

More on the `WHERE` clause in InfluxQL:

* The `WHERE` clause supports comparisons against strings, booleans, floats, integers, and against the `time` of the timestamp.
It supports using regular expressions to match tags, but not to match fields.
* Chain logic together using `AND`  and `OR`, and separate using `(` and `)`.
* Acceptable comparators include:  
`=` equal to  
`<>` not equal to  
`!=` not equal to  
`>` greater than  
`<` less than  
`=~` matches against  
`!~` doesn't match against  

<br>
<br>
# The GROUP BY clause

`GROUP BY` queries group query results by a user-specified tag or time interval.

<table style="width:100%">
  <tr>
    <td><b>GROUP BY tags:</b></td>
    <td><b>GROUP BY time intervals:</b></td>
  </tr>
  <tr>
    <td><a href="#group-by-tags">GROUP BY tags</a></td>
    <td><a href="#basic-syntax">Basic Syntax</a></td>
  </tr>
  <tr>
    <td></td>
    <td><a href="#advanced-syntax">Advanced Syntax</a></td>

  </tr>
  <tr>
    <td></td>
    <td><a href="#group-by-time-intervals-and-fill">GROUP BY time intervals and fill()</a></td>
  </tr>
</table>

## GROUP BY tags

`GROUP BY <tag>` queries group query results by a user-specified tag.

#### Syntax

```
SELECT <function>(<field_key>) FROM <measurement_name> [WHERE <time_range>] GROUP BY [ * | <tag_key>[,<tag_key] ] [fill(<>)]
```

#### Description of Basic Syntax

`GROUP BY <tag>` queries require and InfluxQL [function](/influxdb/v1.0/query_language/functions/).
If the query includes a `WHERE` clause the `GROUP BY` clause must come after the `WHERE` clause.

#### Examples

##### Example 1: Group query results by a single tag
<br>
Calculate the [`MEAN()`](/influxdb/v1.0/query_language/functions/#mean) `water_level` for the different tag values of `location`:
```
> SELECT MEAN("water_level") FROM "h2o_feet" GROUP BY "location"

name: h2o_feet
tags: location=coyote_creek
time			               mean
----			               ----
1970-01-01T00:00:00Z	 5.359342451341401


name: h2o_feet
tags: location=santa_monica
time			               mean
----			               ----
1970-01-01T00:00:00Z	 3.530863470081006
```
>**Note:** In InfluxDB, [epoch 0](https://en.wikipedia.org/wiki/Unix_time) (`1970-01-01T00:00:00Z`) is often used as a null timestamp equivalent.
If you request a query that has no timestamp to return, such as an aggregation function with an unbounded time range, InfluxDB returns epoch 0 as the timestamp.

##### Example 2: Group query results by all tags
<br>
Calculate the [`MEAN()`](/influxdb/v1.0/query_language/functions/#mean) `index` for every tag set in `h2o_quality`:
```
> SELECT MEAN("index") FROM "h2o_quality" GROUP BY *

name: h2o_quality
tags: location=coyote_creek, randtag=1
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.55405446521169


name: h2o_quality
tags: location=coyote_creek, randtag=2
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.49958856271162


name: h2o_quality
tags: location=coyote_creek, randtag=3
time			               mean
----			               ----
1970-01-01T00:00:00Z	 49.5164137518956


name: h2o_quality
tags: location=santa_monica, randtag=1
time			               mean
----			               ----
1970-01-01T00:00:00Z	 50.43829082296367


name: h2o_quality
tags: location=santa_monica, randtag=2
time			               mean
----			               ----
1970-01-01T00:00:00Z	 52.0688508894012


name: h2o_quality
tags: location=santa_monica, randtag=3
time			               mean
----			               ----
1970-01-01T00:00:00Z	 49.29386362086556
```

## GROUP BY time intervals

`GROUP BY time()` queries group query results by a user-specified time interval.

### Basic Syntax

#### Syntax
```
SELECT <function>(<field_key>)
FROM <measurement_name>
WHERE <time_range>
GROUP BY time(<time_interval>),[tag_key] [fill(<fill_option>)]
```

#### Description of Basic Syntax

Basic `GROUP BY time()` queries require an InfluxQL [function](/influxdb/v1.0/query_language/functions/)
and a time range in the `WHERE` clause.
Note that the `GROUP BY` clause must come after the `WHERE` clause.

The `time_interval` in the `GROUP BY time()` clause is a [duration literal](/influxdb/v1.0/query_language/spec/#durations).
It determines how InfluxDB groups query results over time.
For example, a `time_interval` of `5m` groups query results into five-minute
time buckets across the time range specified in the `WHERE` clause.

The `fill(<fill_option>)` is optional.
It changes the value reported for time intervals that have no data.
See [GROUP BY time intervals and `fill()`](#group-by-time-intervals-and-fill)
for additional information.

**Coverage:**

Basic `GROUP BY time()` queries rely on the `time_interval` and on InfluxDB's
preset time boundaries to determine the raw data included in each time interval
and the timestamps returned by the query.

#### Examples of Basic Syntax

The examples below use the following subsample of the sample data:
```
> SELECT "water_level" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'
--------------
time                   water_level   location
2015-08-18T00:00:00Z   8.12          coyote_creek
2015-08-18T00:00:00Z   2.064         santa_monica
2015-08-18T00:06:00Z   8.005         coyote_creek
2015-08-18T00:06:00Z   2.116         santa_monica
2015-08-18T00:12:00Z   7.887         coyote_creek
2015-08-18T00:12:00Z   2.028         santa_monica
2015-08-18T00:18:00Z   7.762         coyote_creek
2015-08-18T00:18:00Z   2.126         santa_monica
2015-08-18T00:24:00Z   7.635         coyote_creek
2015-08-18T00:24:00Z   2.041         santa_monica
2015-08-18T00:30:00Z   7.5           coyote_creek
2015-08-18T00:30:00Z   2.051         santa_monica
```

##### Example 1: Group query results into 12 minute intervals
<br>
[`COUNT()`](/influxdb/v1.0/query_language/functions/#count) the number of
`water_level` points where the `location` equals `coyote_creek` and group results into 12 minute intervals:
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   count
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
```

The result for each timestamp represents a single 12 minute interval.
The count for the first timestamp covers the raw data between `2015-08-18T00:00:00Z`
and up to, but not including, `2015-08-18T00:12:00Z`.
In the raw data above, the relevant field values are `8.12` and `8.005`.
The count for the second timestamp covers the raw data between `2015-08-18T00:12:00Z`
and up to, but not including, `2015-08-18T00:24:00Z.`
In the raw data the relevant field values are `7.887` and `7.762`.

##### Example 2: Group query results into 12 minutes intervals and by a tag key
<br>
[`COUNT()`](/influxdb/v1.0/query_language/functions/#count) the number of
`water_level` points per `location` tag and group results into 12 minute intervals:
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),"location"

name: h2o_feet
tags: location=coyote_creek
time                   count
----                   -----
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2

name: h2o_feet
tags: location=santa_monica
time                   count
----                   -----
2015-08-18T00:00:00Z   2
2015-08-18T00:12:00Z   2
2015-08-18T00:24:00Z   2
```

Note that in the `GROUP BY` clause we separate `time()` and the tag key with a comma.

The query returns two sets of results: one for each tag value of `location`.
The result for each timestamp represents a single 12 minute interval.
The count for the first timestamp covers the raw data between `2015-08-18T00:00:00Z`
and up to, but not including, `2015-08-18T00:12:00Z`.
In the raw data above, the relevant field values are `8.12` and `8.005` for `coyote_creek`
and `2.064` and `2.116` for `santa_monica`.
The count for the second timestamp covers the raw data between `2015-08-18T00:12:00Z`
and up to, but not including, `2015-08-18T00:24:00Z.`
In the raw data the relevant field values are `7.887` and `7.762` for `coyote_creek`
and `2.028` and `2.126` for `santa_monica`.

#### Common Issues with Basic Syntax

##### Issue 1: Unexpected timestamps and values in query results
<br>
With the basic syntax, InfluxDB relies on the `GROUP BY time()` interval
and on the system's preset time boundaries to determine the raw data included
in each time interval and the timestamps returned by the query.
In some cases, this can lead to unexpected results.

**Example**

Raw data:

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:18:00Z'
name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
2015-08-18T00:18:00Z   7.762
```

Query and Results:

The following query groups results by 12 minute intervals, and its time range
covers a single 12 minute interval.

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   mean
2015-08-18T00:00:00Z   1        <----- Note that this timestamp occurs before the start of the query's time range
2015-08-18T00:12:00Z   1
```

Explanation:

InfluxDB has preset time boundaries; when InfluxDB calculates the results for a
single `GROUP BY time()` interval, all raw data must occur both within the
query's time range and within the preset time boundary.
The table below shows the boundary time range, the interval time range, the
points included, and the returned timestamp for each `GROUP BY time()`
interval in the results.

| Time Interval Number | Boundary Time Range | Interval Time Range | Points Included | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 1  | `time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:12:00Z` | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:12:00Z` | `8.005` | `2015-08-18T00:00:00Z` |
| 2  | `time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:24:00Z` | `time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:18:00Z`  | `7.887` | `2015-08-18T00:12:00Z` |

The first preset 12-minute time boundary begins at `00:00` and ends just before
`00:12`.
Only one raw point (`8.005`) falls both within the query's first time interval and in that
first time boundary.
Note that while the returned timestamp occurs before the start of the query's time range,
the query result excludes data that occur before the query's time range.

The second preset 12-minute time boundary begins at `00:12` and ends just before
`00:24`.
Only one raw point (`7.887`) falls both within the query's second time interval and in that
second time boundary.

It is possible to change the default time boundary behavior.
The [advanced `GROUP BY time()` syntax](#advanced-syntax) allows users to shift
the start time of InfluxDB's preset time boundaries.
[Example 3](#example-3-group-query-results-into-12-minute-intervals-and-shift-the-preset-time-boundaries-forward)
in the Advanced Syntax section continues with the query shown in this section;
it shifts forward the preset time boundaries by six minutes such that
InfluxDB returns:

```
name: h2o_feet
--------------
time                   count
2015-08-18T00:06:00Z   2
```

### Advanced Syntax

#### Syntax

```
SELECT <function>(<field_key>)
FROM <measurement_name>
WHERE <time_range>
GROUP BY time(<time_interval>,<offset_interval>),[tag_key] [fill(<fill_option>)]
```

#### Description of Advanced Syntax

See [Description of Basic Syntax](#description-of-basic-syntax) for details
on general syntax and the `time_interval` in the `GROUP BY time()` clause.

The `offset_interval` in the `GROUP BY time()` clause is a
[duration literal](/influxdb/v1.0/query_language/spec/#durations).
It shifts forward or back InfluxDB's preset time boundaries.
The `offset_interval` can be positive or negative.

The `fill(<fill_option>)` is optional.
It changes the value reported for time intervals that have no data.
See [GROUP BY time intervals and `fill()`](#group-by-time-intervals-and-fill)
for additional information.

**Coverage:**

Advanced `GROUP BY time()` queries rely on the `time_interval`, the `offset_interval`
, and on InfluxDB's preset time boundaries to determine the raw data included in each time interval
and the timestamps returned by the query.

#### Examples of Advanced Syntax

The examples below use the following subsample of the sample data:

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:54:00Z'
name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
2015-08-18T00:18:00Z   7.762
2015-08-18T00:24:00Z   7.635
2015-08-18T00:30:00Z   7.5
2015-08-18T00:36:00Z   7.372
2015-08-18T00:42:00Z   7.234
2015-08-18T00:48:00Z   7.11
2015-08-18T00:54:00Z   6.982
```

##### Example 1: Group query results into 18 minute intervals and shift the preset time boundaries forward
<br>
Calculate the [average](/influxdb/v1.0/query_language/functions/#mean) of `water_level`, grouping results into 18 minute
time intervals, and offsetting the preset time boundaries by 6 minutes:

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m,6m)
name: h2o_feet
--------------
time                   mean
2015-08-18T00:06:00Z   7.884666666666667
2015-08-18T00:24:00Z   7.502333333333333
2015-08-18T00:42:00Z   7.108666666666667
```

The offset interval shifts InfluxDB's preset time boundaries forward by six
minutes. The same query without the `offset_interval` returns different
results:

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m)
name: h2o_feet
--------------
time                   mean
2015-08-18T00:00:00Z   7.946
2015-08-18T00:18:00Z   7.6323333333333325
2015-08-18T00:36:00Z   7.238666666666667
2015-08-18T00:54:00Z   6.982
```

The time boundaries and returned timestamps for the query **without** the
`offset_interval` adhere to InfluxDB's preset time boundaries:

| Time Interval Number | Boundary Time Range |  Interval Time Range | Points Included | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 1  | `time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:18:00Z` | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:18:00Z` | `8.005`,`7.887` | `2015-08-18T00:00:00Z` |
| 2  | `time >= 2015-08-18T00:18:00Z AND time < 2015-08-18T00:36:00Z` | <--- same | `7.762`,`7.635`,`7.5` | `2015-08-18T00:18:00Z` |
| 3  | `time >= 2015-08-18T00:36:00Z AND time < 2015-08-18T00:54:00Z` | <--- same | `7.372`,`7.234`,`7.11` | `2015-08-18T00:36:00Z` |
| 4  | `time >= 2015-08-18T00:54:00Z AND time < 2015-08-18T01:12:00Z` | `time = 2015-08-18T00:54:00Z` | `6.982` | `2015-08-18T00:54:00Z` |

The first preset 18-minute time boundary begins at `00:00` and ends just before
`00:18`.
Two raw points (`8.005` and `7.887`) fall both within the first interval's time range and in that
first time boundary.
Note that while the returned timestamp occurs before the start of the query's time range,
the query result excludes data that occur before the query's time range.

The second preset 18-minute time boundary begins at `00:18` and ends just before
`00:36`.
Three raw points (`7.762` and `7.635` and `7.5`) fall both within the second interval's time range and in that
second time boundary. In this case, the boundary time range and the interval's time range are the same.

The fourth preset 18-minute time boundary begins at `00:54` and ends just before
`1:12:00`.
One raw point (`6.982`) falls both within the fourth interval's time range and in that
fourth time boundary.

The time boundaries and returned timestamps for the query **with** the
`offset_interval` adhere to InfluxDB's preset time boundaries and the six-minute
shift forward:

| Time Interval Number | Boundary Time Range | Interval Time Range | Points Includes | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | ------------- |
| 1  | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:24:00Z` | <--- same | `8.005`,`7.887`,`7.762` | `2015-08-18T00:06:00Z` |
| 2  | `time >= 2015-08-18T00:24:00Z AND time < 2015-08-18T00:42:00Z` | <--- same | `7.635`,`7.5`,`7.372` | `2015-08-18T00:24:00Z` |
| 3  | `time >= 2015-08-18T00:42:00Z AND time < 2015-08-18T01:00:00Z` | <--- same | `7.234`,`7.11`,`6.982` | `2015-08-18T00:42:00Z` |

The six-minute offset interval shifts forward the preset boundary's time range
such that the preset boundary time range and the interval time range are the
same.

With the offset, each interval performs the calculation on three points, and
the timestamp returned matches both the start of the boundary time range and the
start of the interval time range.

##### Example 2: Group query results into 12 minute intervals and shift the preset time boundaries back
<br>
Calculate the [average](/influxdb/v1.0/query_language/functions/#mean) of `water_level`, grouping results into 18 minute
time intervals, and offsetting the preset time boundaries by -12 minutes:

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m,-12m)
name: h2o_feet
--------------
time                   mean
2015-08-18T00:06:00Z   7.884666666666667
2015-08-18T00:24:00Z   7.502333333333333
2015-08-18T00:42:00Z   7.108666666666667
```

The offset interval shifts InfluxDB's preset time boundaries back by 12
minutes. The same query without the `offset_interval` returns different
results:

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time <= '2015-08-18T00:54:00Z' GROUP BY time(18m)
name: h2o_feet
--------------
time                   mean
2015-08-18T00:00:00Z   7.946
2015-08-18T00:18:00Z   7.6323333333333325
2015-08-18T00:36:00Z   7.238666666666667
2015-08-18T00:54:00Z   6.982
```

The time boundaries and returned timestamps for the query **without** the
`offset_interval` adhere to InfluxDB's preset time boundaries:

| Time Interval Number | Boundary Time Range |  Interval Time Range | Points Included | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 1  | `time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:18:00Z` | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:18:00Z` | `8.005`,`7.887` | `2015-08-18T00:00:00Z` |
| 2  | `time >= 2015-08-18T00:18:00Z AND time < 2015-08-18T00:36:00Z` | <--- same | `7.762`,`7.635`,`7.5` | `2015-08-18T00:18:00Z` |
| 3  | `time >= 2015-08-18T00:36:00Z AND time < 2015-08-18T00:54:00Z` | <--- same | `7.372`,`7.234`,`7.11` | `2015-08-18T00:36:00Z` |
| 4  | `time >= 2015-08-18T00:54:00Z AND time < 2015-08-18T01:12:00Z` | `time = 2015-08-18T00:54:00Z` | `6.982` | `2015-08-18T00:54:00Z` |

The first preset 18-minute time boundary begins at `00:00` and ends just before
`00:18`.
Two raw points (`8.005` and `7.887`) fall both within the first interval's time range and in that
first time boundary.
Note that while the returned timestamp occurs before the start of the query's time range,
the query result excludes data that occur before the query's time range.

The second preset 18-minute time boundary begins at `00:18` and ends just before
`00:36`.
Three raw points (`7.762` and `7.635` and `7.5`) fall both within the second interval's time range and in that
second time boundary. In this case, the boundary time range and the interval's time range are the same.

The fourth preset 18-minute time boundary begins at `00:54` and ends just before
`1:12:00`.
One raw point (`6.982`) falls both within the fourth interval's time range and in that
fourth time boundary.

The time boundaries and returned timestamps for the query **with** the
`offset_interval` adhere to InfluxDB's preset time boundaries and the -12-minute
shift forward:

| Time Interval Number | Boundary Time Range | Query Time Range | Points Includes | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | ------------- |
| 1  | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:24:00Z` | <--- same | `8.005`,`7.887`,`7.762` | `2015-08-18T00:06:00Z` |
| 2  | `time >= 2015-08-18T00:24:00Z AND time < 2015-08-18T00:42:00Z` | <--- same | `7.635`,`7.5`,`7.372` | `2015-08-18T00:24:00Z` |
| 3  | `time >= 2015-08-18T00:42:00Z AND time < 2015-08-18T01:00:00Z` | <--- same | `7.234`,`7.11`,`6.982` | `2015-08-18T00:42:00Z` |

The negative six-minute offset interval shifts back the preset boundary's time range
such that the preset boundary time range and the interval time range are the
same.

With the offset, each interval performs the calculation on three points, and
the timestamp returned matches both the start of the boundary time range and the
start of the interval time range.

##### Example 3: Group query results into 12 minute intervals and shift the preset time boundaries forward
<br>
This example is a continuation of the scenario outlined in [Common Issues with Basic Syntax](#common-issues-with-basic-syntax).

[`COUNT()`](/influxdb/v1.0/query_language/functions/#count) the number of
`water_level`, grouping results into 12-minute time intervals, and offsetting the
preset time boundaries by six minutes:

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m,6m)

name: h2o_feet
--------------
time                   count
2015-08-18T00:06:00Z   2
```

The offset interval shifts InfluxDB's preset time boundaries forward by six
minutes.
The same query without the `offset_interval` returns different results:

```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   mean
2015-08-18T00:00:00Z   1
2015-08-18T00:12:00Z   1
```

The time boundaries and returned timestamps for the query **without** the
`offset_interval` adhere to InfluxDB's preset time boundaries:

| Time Interval Number | Boundary Time Range | Query Time Range | Points Included | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 1  | `time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:12:00Z` | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:12:00Z` | `8.005` | `2015-08-18T00:00:00Z` |
| 2  | `time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:24:00Z` | `time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:18:00Z`  | `7.887` | `2015-08-18T00:12:00Z` |

The first preset 12-minute time boundary begins at `00:00` and ends just before
`00:12`.
Only one raw point (`8.005`) falls both within the query's first time interval and in that
first time boundary.
Note that while the returned timestamp occurs before the start of the query's time range,
the query result excludes data that occur before the query's time range.

The second preset 12-minute time boundary begins at `00:12` and ends just before
`00:24`.
Only one raw point (`7.887`) falls both within the query's second time interval and in that
second time boundary.

The time boundaries and returned timestamps for the query **with** the
`offset_interval` adhere to InfluxDB's preset time boundaries and the six-minute
shift forward:

| Time Interval Number | Boundary Time Range | Query Time Range | Points Included | Returned Timestamp |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 1  | `time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:24:00Z` | <--- same | `8.005`,`7.887` | `2015-08-18T00:06:00Z` |

The six-minute offset interval shifts forward the preset boundary's time range
such that the preset boundary time range and the interval time range are the
same.
With the offset, the query returns a single result, and the timestamp returned
matches both the start of the boundary time range and the start of the interval
time range.

## `GROUP BY` time intervals and `fill()`

`fill()` changes the value reported for time intervals that have no data.

#### Syntax

```
SELECT <function>(<field_key>)
FROM <measurement_name>
WHERE <time_range>
GROUP BY time(time_interval,[<offset_interval]),[tag_key] [fill(<fill_option>)]
```

#### Description of Syntax

By default, a `GROUP BY time()` interval with no data reports `null` as its
value in the output column.
`fill()` changes the value reported for time intervals that have no data.
Note that `fill()` must go at the end of the `GROUP BY` clause if you're
`GROUP(ing) BY` several things (for example, both tags and a time interval).

##### fill_option
<br>

Any numerical value
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Reports any numerical value for time intervals with no data.

`none`
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Reports no timestamp and no value for time intervals with no data.

`null`
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Reports `null` for time intervals with no data. This is the same as the default behavior.

`previous`
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;
Reports the value from the previous time interval for time intervals with no data.

#### Examples

{{< vertical-tabs >}}
{{% tabs %}}
[Example 1: fill(100)](#)
[Example 2: fill(none)](#)
[Example 3: fill(null)](#)
[Example 4: fill(previous)](#)
{{% /tabs %}}
{{< tab-content-container >}}

{{% tab-content %}}

Without `fill(100)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   
```

With `fill(100)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(100)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   100
```

`fill(100)` changes the value reported for the time interval with no data to `100`.

{{% /tab-content %}}

{{% tab-content %}}

Without `fill(none)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z
```

With `fill(none)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(none)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
```

`fill(none)` reports no value and no timestamp for the time interval with no data.

{{% /tab-content %}}

{{% tab-content %}}

Without `fill(null)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z
```

With `fill(null)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(null)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   
```

`fill(null)` reports `null` as the value for the time interval with no data.
That result matches the result of the query without `fill(null)`.

{{% /tab-content %}}

{{% tab-content %}}

Without `fill(previous)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z
```

With `fill(previous)`:
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-09-18T16:00:00Z' AND time <= '2015-09-18T16:42:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:00:00Z   3.599
2015-09-18T16:12:00Z   3.402
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   3.235
```

`fill(previous)` changes the value reported for the time interval with no data to `3.235`,
the value from the previous time interval.

{{% /tab-content %}}

{{< /tab-content-container >}}
{{< /vertical-tabs >}}

#### Common issues with `fill()`

##### Issue 1: `fill()` when no data fall within the query's time range
<br>
Currently, queries ignore `fill()` if no data fall within the query's time range.
This is the expected behavior. An open
[feature request](https://github.com/influxdata/influxdb/issues/6967) on GitHub
proposes that `fill()` should force a return of values even if the query's time
range covers no data.

**Example**

The following query returns no data because `water_level` has no points within
the query's time range.
Note that `fill(800)` has no effect on the query results.
```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-09-18T22:00:00Z' AND time <= '2015-09-18T22:18:00Z' GROUP BY time(12m) fill(800)
>
```

##### Issue 2: `fill(previous)` when the previous result falls outside the query's time range
<br>
`fill(previous)` doesn’t fill the result for a time interval if the previous
value is outside the query’s time range.

**Example**

The following query covers the time range between `2015-09-18T16:24:00Z` and `2015-09-18T16:54:00Z`.
Note that `fill(previous)` fills the result for `2015-09-18T16:36:00Z` with the
result from `2015-09-18T16:24:00Z`.
```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE location = 'coyote_creek' AND time >= '2015-09-18T16:24:00Z' AND time <= '2015-09-18T16:54:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:24:00Z   3.235
2015-09-18T16:36:00Z   3.235
2015-09-18T16:48:00Z   4
```

The next query shortens the time range in the previous query.
It now covers the time between `2015-09-18T16:36:00Z` and `2015-09-18T16:54:00Z`.
Note that `fill(previous)` does not fill the result for `2015-09-18T16:36:00Z` with the
result from `2015-09-18T16:24:00Z`; the result for `2015-09-18T16:24:00Z` is outside the query's
shorter time range.

```
> SELECT MAX("water_level") FROM "h2o_feet" WHERE location = 'coyote_creek' AND time >= '2015-09-18T16:36:00Z' AND time <= '2015-09-18T16:54:00Z' GROUP BY time(12m) fill(previous)

name: h2o_feet
--------------
time                   max
2015-09-18T16:36:00Z
2015-09-18T16:48:00Z   4
```

<br>
<br>
# The INTO clause
## Relocate data
Copy data to another database, retention policy, and measurement with the `INTO` clause:
```sql
SELECT <field_key> INTO <different_measurement> FROM <current_measurement> [WHERE <stuff>] [GROUP BY <stuff>]
```

Write the field `water_level` in `h2o_feet` to a new measurement (`h2o_feet_copy`) in the same database:
```sql
> SELECT "water_level" INTO "h2o_feet_copy" FROM "h2o_feet" WHERE "location" = 'coyote_creek'
```

The CLI response shows the number of points that InfluxDB wrote to `h2o_feet_copy`:
```
name: result
------------
time			               written
1970-01-01T00:00:00Z	 7604
```

Write the field `water_level` in `h2o_feet` to a new measurement (`h2o_feet_copy`) and to the retention policy `autogen` in the [already-existing](/influxdb/v1.0/query_language/database_management/#create-a-database-with-create-database) database `where_else`:
```sql
> SELECT "water_level" INTO "where_else"."autogen"."h2o_feet_copy" FROM "h2o_feet" WHERE "location" = 'coyote_creek'
```

CLI response:
```bash
name: result
------------
time			               written
1970-01-01T00:00:00Z	 7604
```

> **Note**: If you use `SELECT *` with `INTO`, the query converts tags in the current measurement to fields in the new measurement.
This can cause InfluxDB to overwrite points that were previously differentiated by a tag value.
Use `GROUP BY <tag_key>` to preserve tags as tags.

## Downsample data
Combine the `INTO` clause with an InfluxQL [function](/influxdb/v1.0/query_language/functions/) and a `GROUP BY` clause to write the lower precision query results to a different measurement:
```sql
SELECT <function>(<field_key>) INTO <different_measurement> FROM <current_measurement> WHERE <stuff> GROUP BY <stuff>
```

> **Note:** The `INTO` queries in this section downsample old data, that is, data that have already been written to InfluxDB.
If you want InfluxDB to automatically query and downsample all future data see [Continuous Queries](/influxdb/v1.0/query_language/continuous_queries/).

Calculate the average `water_level` in `santa_monica`, and write the results to a new measurement (`average`) in the same database:
```sql
> SELECT MEAN("water_level") INTO "average" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
```

The CLI response shows the number of points that InfluxDB wrote to the new measurement:
```bash
name: result
------------
time			               written
1970-01-01T00:00:00Z	 3
```

To see the query results, select everything from the new measurement `average` in `NOAA_water_database`:
```bash
> SELECT * FROM "average"
name: average
-------------
time			               mean
2015-08-18T00:00:00Z	 2.09
2015-08-18T00:12:00Z	 2.077
2015-08-18T00:24:00Z	 2.0460000000000003
```

Calculate the average `water_level` and the max `water_level` in `santa_monica`, and write the results to a new measurement (`aggregates`) in a different database (`where_else`):
```sql
> SELECT MEAN("water_level"), max("water_level") INTO "where_else"."autogen"."aggregates" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
```

CLI response:
```bash
name: result
------------
time			               written
1970-01-01T00:00:00Z	 3
```

Select everything from the new measurement `aggregates` in the database `where_else`:
```bash
> SELECT * FROM "where_else"."autogen"."aggregates"
name: aggregates
----------------
time			               max	   mean
2015-08-18T00:00:00Z	 2.116	 2.09
2015-08-18T00:12:00Z	 2.126	 2.077
2015-08-18T00:24:00Z	 2.051	 2.0460000000000003
```

Calculate the average `degrees` for all temperature measurements (`h2o_temperature` and `average_temperature`) in the `NOAA_water_database` and write the results to new measurements with the same names in a different database (`where_else`).
`:MEASUREMENT` tells InfluxDB to write the query results to measurements with the same names as those targeted by the query:
```sql
> SELECT MEAN("degrees") INTO "where_else"."autogen".:MEASUREMENT FROM /temperature/ WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
```

CLI response:
```bash
name: result
------------
time			               written
1970-01-01T00:00:00Z	 6
```

Select the `mean` field from all new temperature measurements in the database `where_else`.
```bash
> SELECT "mean" FROM "where_else"."autogen"./temperature/
name: average_temperature
-------------------------
time			               mean
2015-08-18T00:00:00Z	 78.5
2015-08-18T00:12:00Z	 84
2015-08-18T00:24:00Z	 74.75

name: h2o_temperature
---------------------
time			                mean
2015-08-18T00:00:00Z	  63.75
2015-08-18T00:12:00Z	  63.5
2015-08-18T00:24:00Z	  63.5
```

More on downsampling with `INTO`:

* InfluxDB does not store null values.
Depending on the frequency of your data, the query results may be missing time intervals.
Use [fill()](/influxdb/v1.0/query_language/data_exploration/#the-group-by-clause-and-fill) to ensure that every time interval appears in the results.
* The number of writes in the CLI response includes one write for every time interval in the query's time range even if there is no data for some of the time intervals.

<br>
<br>
# Limit query returns with LIMIT and SLIMIT
InfluxQL supports two different clauses to limit your query results:

* `LIMIT <N>` returns the first \<N> [points](/influxdb/v1.0/concepts/glossary/#point) from each [series](/influxdb/v1.0/concepts/glossary/#series) in the specified measurement.
* `SLIMIT <N>` returns every point from \<N> series in the specified measurement.
* `LIMIT <N>` followed by `SLIMIT <N>` returns the first \<N> points from \<N> series in the specified measurement.

## Limit the number of results returned per series with `LIMIT`
Use `LIMIT <N>` with `SELECT` and `GROUP BY *` to return the first \<N> points from each series.

Return the three oldest points from each series associated with the measurement `h2o_feet`:
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3
```

CLI response:
```bash
name: h2o_feet
tags: location=coyote_creek
time			              water_level
----			              -----------
2015-08-18T00:00:00Z	8.12
2015-08-18T00:06:00Z	8.005
2015-08-18T00:12:00Z	7.887

name: h2o_feet
tags: location=santa_monica
time			              water_level
----			              -----------
2015-08-18T00:00:00Z	2.064
2015-08-18T00:06:00Z	2.116
2015-08-18T00:12:00Z	2.028
```

> **Note:** If \<N> is greater than the number of points in the series, InfluxDB returns all points in the series.

## Limit the number of series returned with `SLIMIT`
Use `SLIMIT <N>` with `SELECT` and `GROUP BY *` to return every point from \<N> series.

Return everything from one of the series associated with the measurement `h2o_feet`:
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY * SLIMIT 1
```

CLI response:
```bash
name: h2o_feet
tags: location=coyote_creek
time			              water_level
----			              -----
2015-08-18T00:00:00Z	8.12
2015-08-18T00:06:00Z	8.005
2015-08-18T00:12:00Z	7.887
[...]
2015-09-18T16:12:00Z	3.402
2015-09-18T16:18:00Z	3.314
2015-09-18T16:24:00Z	3.235
```

> **Note:** If \<N> is greater than the number of series associated with the specified measurement, InfluxDB returns all points from every series.

## Limit the number of points and series returned with `LIMIT` and `SLIMIT`
Use `LIMIT <N1>` followed by `SLIMIT <N2>` with `GROUP BY *` to return \<N1> points from \<N2> series.

Return the three oldest points from one of the series associated with the measurement `h2o_feet`:
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3 SLIMIT 1
```

CLI response:
```bash
name: h2o_feet
tags: location=coyote_creek
time			               water_level
----			               -----------
2015-08-18T00:00:00Z	 8.12
2015-08-18T00:06:00Z	 8.005
2015-08-18T00:12:00Z	 7.887
```

> **Note:** If \<N1> is greater than the number of points in the series, InfluxDB returns all points in the series.
If \<N2> is greater than the number of series associated with the specified measurement, InfluxDB returns points from every series.

<br>
<br>
# Sort query returns with ORDER BY time DESC
By default, InfluxDB returns results in ascending time order - so the first points that are returned are the oldest points by timestamp.
Use `ORDER BY time DESC` to see the newest points by timestamp.

Return the oldest five points from one series **without** `ORDER BY time DESC`:  
```sql
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' LIMIT 5
```

CLI response:  
```bash
name: h2o_feet
----------
time			water_level
2015-08-18T00:00:00Z	2.064
2015-08-18T00:06:00Z	2.116
2015-08-18T00:12:00Z	2.028
2015-08-18T00:18:00Z	2.126
2015-08-18T00:24:00Z	2.041
```

Now include  `ORDER BY time DESC` to get the newest five points from the same series:  
```sql
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' ORDER BY time DESC LIMIT 5
```

CLI response:  
```bash
name: h2o_feet
----------
time			water_level
2015-09-18T21:42:00Z	4.938
2015-09-18T21:36:00Z	5.066
2015-09-18T21:30:00Z	5.01
2015-09-18T21:24:00Z	5.013
2015-09-18T21:18:00Z	5.072
```

Finally, use `GROUP BY` with `ORDER BY time DESC` to return the last five points from each series:  
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY "location" ORDER BY time DESC LIMIT 5
```

CLI response:
```bash
name: h2o_feet
tags: location=santa_monica
time			               water_level
----			               -----------
2015-09-18T21:42:00Z	 4.938
2015-09-18T21:36:00Z	 5.066
2015-09-18T21:30:00Z	 5.01
2015-09-18T21:24:00Z	 5.013
2015-09-18T21:18:00Z	 5.072

name: h2o_feet
tags: location=coyote_creek
time			               water_level
----			               -----------
2015-09-18T16:24:00Z	 3.235
2015-09-18T16:18:00Z	 3.314
2015-09-18T16:12:00Z	 3.402
2015-09-18T16:06:00Z	 3.497
2015-09-18T16:00:00Z	 3.599
```

<br>
<br>
# Paginate query returns with OFFSET and SOFFSET

## Use `OFFSET` to paginate the results returned
For example, get the first three points written to a series:

```sql
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'coyote_creek' LIMIT 3
```

CLI response:  
```bash
name: h2o_feet
----------
time			               water_level
2015-08-18T00:00:00Z	 8.12
2015-08-18T00:06:00Z	 8.005
2015-08-18T00:12:00Z	 7.887
```

Then get the second three points from that same series:

```sql
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'coyote_creek' LIMIT 3 OFFSET 3
```

CLI response:
```bash
name: h2o_feet
----------
time			               water_level
2015-08-18T00:18:00Z	 7.762
2015-08-18T00:24:00Z	 7.635
2015-08-18T00:30:00Z	 7.5
```

## Use `SOFFSET` to paginate the series returned
For example, get the first three points from a single series:
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3 SLIMIT 1
```

CLI response:
```
name: h2o_feet
tags: location=coyote_creek
time			               water_level
----			               -----------
2015-08-18T00:00:00Z	 8.12
2015-08-18T00:06:00Z	 8.005
2015-08-18T00:12:00Z	 7.887
```

Then get the first three points from the next series:
```sql
> SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3 SLIMIT 1 SOFFSET 1
```

CLI response:
```
name: h2o_feet
tags: location=santa_monica
time			               water_level
----			               -----------
2015-08-18T00:00:00Z	 2.064
2015-08-18T00:06:00Z	 2.116
2015-08-18T00:12:00Z	 2.028
```

<br>
<br>
# Multiple statements in queries
Separate multiple statements in a query with a semicolon.
For example:
<br>
<br>
```sql
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time > now() - 2w GROUP BY "location",time(24h) fill(none); SELECT COUNT("water_level") FROM "h2o_feet" WHERE time > now() - 2w GROUP BY "location",time(24h) fill(80)
```

<br>
<br>
# Merge series in queries
In InfluxDB, queries merge series automatically.

The `NOAA_water_database` database has two [series](/influxdb/v1.0/concepts/glossary/#series).
The first series is made up of the measurement `h2o_feet` and the tag key `location` with the tag value `coyote_creek`.
The second series is made of up the measurement `h2o_feet` and the tag key `location` with the tag value `santa_monica`.

The following query automatically merges those two series when it calculates the [average](/influxdb/v1.0/query_language/functions/#mean) `water_level`:

```sql
> SELECT MEAN("water_level") FROM "h2o_feet"
```

CLI response:
```bash
name: h2o_feet
--------------
time			               mean
1970-01-01T00:00:00Z	 4.442107025822521
```

If you only want the `MEAN()` `water_level` for the first series, specify the tag set in the `WHERE` clause:
```sql
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" = 'coyote_creek'
```

CLI response:
```bash
name: h2o_feet
--------------
time			               mean
1970-01-01T00:00:00Z	 5.359342451341401
```

> **NOTE:** In InfluxDB, [epoch 0](https://en.wikipedia.org/wiki/Unix_time) (`1970-01-01T00:00:00Z`) is often used as a null timestamp equivalent.
If you request a query that has no timestamp to return, such as an aggregation function with an unbounded time range, InfluxDB returns epoch 0 as the timestamp.

<br>
<br>
# Time syntax in queries  
InfluxDB is a time series database so, unsurprisingly, InfluxQL has a lot to do with specifying time ranges.
If you do not specify start and end times in your query, they default to epoch 0 (`1970-01-01T00:00:00Z`) and `now()`.
The following sections detail how to specify different start and end times in queries.

## Relative time
`now()` is the Unix time of the server at the time the query is executed on that server.
Use `now()` to calculate a timestamp relative to the server's
current timestamp.

Query data starting an hour ago and ending `now()`:
```sql
> SELECT "water_level" FROM "h2o_feet" WHERE time > now() - 1h
```

Query data that occur between epoch 0 and 1,000 days from `now()`:  
```sql
> SELECT "level description" FROM "h2o_feet" WHERE time < now() + 1000d
```

* Note the whitespace between the operator and the time duration.
Leaving that whitespace out can cause InfluxDB to return no results or an `error parsing query` error .

The other options for specifying time durations with `now()` are listed below.  
`u` microseconds  
`ms` milliseconds  
`s` seconds  
`m` minutes  
`h` hours  
`d` days  
`w` weeks   

## Absolute time
#### Date time strings
Specify time with date time strings.
Date time strings can take two formats: `YYYY-MM-DD HH:MM:SS.nnnnnnnnn` and  `YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ`, where the second specification is [RFC3339](https://www.ietf.org/rfc/rfc3339.txt).
Nanoseconds (`nnnnnnnnn`) are optional in both formats.

Examples:

Query data between August 18, 2015 23:00:01.232000000 and September 19, 2015 00:00:00 with the timestamp syntax `YYYY-MM-DD HH:MM:SS.nnnnnnnnn` and  `YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ`:

```sql
> SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-08-18 23:00:01.232000000' AND time < '2015-09-19'
```
```sql
> SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-08-18T23:00:01.232000000Z' AND time < '2015-09-19'
```

Query data that occur 6 minutes after September 18, 2015 21:24:00:

```sql
> SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-09-18T21:24:00Z' + 6m
```

Things to note about querying with date time strings:

* Single quote the date time string.
InfluxDB returns as error (`ERR: invalid operation: time and *influxql.VarRef are not compatible`) if you double quote the date time string.
* If you only specify the date, InfluxDB sets the time to `00:00:00`.

### Epoch time
Specify time with timestamps in epoch time.
Epoch time is the number of nanoseconds that have elapsed since 00:00:00 Coordinated Universal Time (UTC), Thursday, 1 January 1970.
Indicate the units of the timestamp at the end of the timestamp (see the section above for a list of acceptable time units).

Examples:

Return all points that occur after  `2014-01-01 00:00:00`:  
```sql
> SELECT * FROM "h2o_feet" WHERE time > 1388534400s
```

Return all points that occur 6 minutes after `2015-09-18 21:24:00`:

```sql
> SELECT * FROM "h2o_feet" WHERE time > 24043524m + 6m
```

> **Note**: Currently, InfluxDB does not support using `OR` with absolute time
in the `WHERE` clause.
See [Frequently Asked Questions](/influxdb/v1.0/troubleshooting/frequently-asked-questions/#why-is-my-query-with-a-where-or-time-clause-returning-empty-results)
for more information.

<br>
<br>
# Data types and cast operations in queries

## Data types

Field values can be floats, integers, strings, or booleans.
Specify the relevant field type in a `SELECT` query with the syntax
`<field_key>::<type>`.

Example:

Specify that the response should contain floats from the field
`water_level`:
```
> SELECT "water_level"::float FROM "h2o_feet" LIMIT 4
name: h2o_feet
--------------
time		                	water_level
2015-08-18T00:00:00Z	  8.12
2015-08-18T00:00:00Z	  2.064
2015-08-18T00:06:00Z	  8.005
2015-08-18T00:06:00Z	  2.116
```

> **Note:**  It is possible for field value types to differ across shards.
Please see
[Frequently Asked Questions](/influxdb/v1.0/troubleshooting/frequently-asked-questions/#how-does-influxdb-handle-field-type-discrepancies-across-shards)
for more information on how InfluxDB handles field value type discrepancies.

## Cast Operations
The `<field_key>::<type>` syntax supports casting field values from integers to
floats or from floats to integers.
Attempting to cast a float or integer to a string or boolean yields an empty
response.

Examples:

Specify that the response should contain integers from the field `water_level`:
```
> SELECT "water_level"::integer FROM "h2o_feet" LIMIT 4
name: h2o_feet
--------------
time		                	water_level
2015-08-18T00:00:00Z	  8
2015-08-18T00:00:00Z	  2
2015-08-18T00:06:00Z	  8
2015-08-18T00:06:00Z	  2
```

Specify that the response should contain strings from the field `water_level`
(this functionality is not supported):
```
> SELECT "water_level"::string FROM "h2o_feet" LIMIT 4
>
```

<br>
<br>
# Regular expressions in queries

Regular expressions are surrounded by `/` characters and use [Golang's regular expression syntax](http://golang.org/pkg/regexp/syntax/).
Use regular expressions when selecting measurements and tags.

> **Note:** You cannot use regular expressions to match databases, retention policies, or fields.
You can only use regular expressions to match measurements and tags.

In this section we'll be using all of the measurements in the [sample data](/influxdb/v1.0/query_language/data_exploration/#sample-data):
`h2o_feet`, `h2o_quality`, `h2o_pH`, `average_temperature`, and `h2o_temperature`.
Please note that every measurement besides `h2o_feet` is fictional and contains fictional data.

## Regular expressions and selecting measurements
Select the oldest point from every measurement in the `NOAA_water_database` database:
```sql
> SELECT * FROM /.*/ LIMIT 1
```

CLI response:
```bash
name: average_temperature
-------------------------
time			               degrees	 index	 level description	    location	     pH	 randtag	 water_level
2015-08-18T00:00:00Z	 82					                               coyote_creek


name: h2o_feet
--------------
time			               degrees	 index	 level description	    location	     pH	 randtag	 water_level
2015-08-18T00:00:00Z			               between 6 and 9 feet	 coyote_creek			            8.12


name: h2o_pH
------------
time			               degrees	 index	 level description	    location	     pH	 randtag	 water_level
2015-08-18T00:00:00Z						                                  coyote_creek	 7


name: h2o_quality
-----------------
time			               degrees	 index	 level description	    location	     pH	 randtag	 water_level
2015-08-18T00:00:00Z		         41				                       coyote_creek		    1


name: h2o_temperature
---------------------
time			               degrees	 index	 level description	    location	     pH	 randtag	 water_level
2015-08-18T00:00:00Z	 60					                               coyote_creek
```

* Alternatively, `SELECT` all of the measurements in `NOAA_water_database` by typing them out and separating each name with a comma , but that could get tedious:

    ```sql
> SELECT * FROM "average_temperature","h2o_feet","h2o_pH","h2o_quality","h2o_temperature" LIMIT 1
    ```

Select the first three points from every measurement whose name starts with `h2o`:  
```sql
> SELECT * FROM /^h2o/ LIMIT 3
```
CLI response:  
```bash
name: h2o_feet
--------------
time			               degrees	 index	 level description	    location	     pH	randtag	water_level
2015-08-18T00:00:00Z			               between 6 and 9 feet	 coyote_creek			          8.12
2015-08-18T00:00:00Z			               below 3 feet		        santa_monica			          2.064
2015-08-18T00:06:00Z			               between 6 and 9 feet	 coyote_creek			          8.005


name: h2o_pH
------------
time			               degrees	 index	 level description	    location	     pH	randtag	water_level
2015-08-18T00:00:00Z						                                  coyote_creek	 7
2015-08-18T00:00:00Z						                                  santa_monica	 6
2015-08-18T00:06:00Z						                                  coyote_creek	 8


name: h2o_quality
-----------------
time			               degrees	 index	 level description	    location	     pH	randtag	water_level
2015-08-18T00:00:00Z		         99				                       santa_monica		   2
2015-08-18T00:00:00Z		         41				                       coyote_creek		   1
2015-08-18T00:06:00Z		         11				                       coyote_creek		   3


name: h2o_temperature
---------------------
time			               degrees	 index	 level description	    location	     pH	randtag	water_level
2015-08-18T00:00:00Z	 70					                               santa_monica
2015-08-18T00:00:00Z	 60					                               coyote_creek
2015-08-18T00:06:00Z	 60					                               santa_monica

```

Select the first 5 points from every measurement whose name contains `temperature`:

```sql
> SELECT * FROM /.*temperature.*/ LIMIT 5
```

CLI response:
```bash
name: average_temperature
-------------------------
time			              degrees	location
2015-08-18T00:00:00Z	85	     santa_monica
2015-08-18T00:00:00Z	82	     coyote_creek
2015-08-18T00:06:00Z	73	     coyote_creek
2015-08-18T00:06:00Z	74	     santa_monica
2015-08-18T00:12:00Z	86	     coyote_creek

name: h2o_temperature
---------------------
time			              degrees	location
2015-08-18T00:00:00Z	60	     coyote_creek
2015-08-18T00:00:00Z	70	     santa_monica
2015-08-18T00:06:00Z	65	     coyote_creek
2015-08-18T00:06:00Z	60	     santa_monica
2015-08-18T00:12:00Z	68	     coyote_creek
```

## Regular expressions and specifying tags
Use regular expressions to specify tags in the `WHERE` clause.
The relevant comparators include:  
`=~` matches against  
`!~` doesn't match against

Select the oldest four points from the measurement `h2o_feet` where the value of the tag `location` does not include an `a`:
```sql
> SELECT * FROM "h2o_feet" WHERE "location" !~ /.*a.*/ LIMIT 4
```

CLI response:
```bash
name: h2o_feet
--------------
time			               level description	    location	     water_level
2015-08-18T00:00:00Z	 between 6 and 9 feet 	coyote_creek	 8.12
2015-08-18T00:06:00Z	 between 6 and 9 feet	 coyote_creek	 8.005
2015-08-18T00:12:00Z	 between 6 and 9 feet	 coyote_creek	 7.887
2015-08-18T00:18:00Z	 between 6 and 9 feet	 coyote_creek	 7.762
```

Select the oldest four points from the measurement `h2o_feet` where the value of the tag `location` includes a `y` or an `m` and `water_level` is greater than zero:
```sql
> SELECT * FROM "h2o_feet" WHERE ("location" =~ /.*y.*/ OR "location" =~ /.*m.*/) AND "water_level" > 0 LIMIT 4
```
or
<br>
<br>
```sql
> SELECT * FROM "h2o_feet" WHERE "location" =~ /[ym]/ AND "water_level" > 0 LIMIT 4
```

CLI response:
```
name: h2o_feet
--------------
time			               level description	    location	     water_level
2015-08-18T00:00:00Z	 between 6 and 9 feet	 coyote_creek	 8.12
2015-08-18T00:00:00Z	 below 3 feet		        santa_monica	 2.064
2015-08-18T00:06:00Z	 between 6 and 9 feet	 coyote_creek	 8.005
2015-08-18T00:06:00Z	 below 3 feet		        santa_monica	 2.116
```

See [the WHERE clause](/influxdb/v1.0/query_language/data_exploration/#the-where-clause) section for an example of how to return data where a tag key has a value and an example of how to return data where a tag key has no value using regular expressions.
