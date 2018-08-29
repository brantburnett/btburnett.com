---
id: 305
title: Announcing Linq2Couchbase 1.4
date: 2018-08-29T09:00:00-04:00
categories:
  - Couchbase
---
I am pleased to announce the release of [Linq2Couchbase 1.4](https://www.nuget.org/packages/Linq2Couchbase/1.4.0). This version includes many new features and improvements.

## ANSI Joins with Couchbase Server 5.5

[Couchbase Server 5.5](https://blog.couchbase.com/announcing-couchbase-server-5-5/) adds support for full ANSI join between documents. Previously, it was only possible to join when at least one side used a document key. With ANSI joins, it's possible to join using any indexed attribute.

Linq2Couchbase 1.4 [adds ANSI joins to LINQ queries](https://github.com/couchbaselabs/Linq2Couchbase/blob/master/docs/joins.md#ansi-joins), so long as you're connected to a Couchbase 5.5 server. Just use standard LINQ syntax and the magic happens!

```cs
var context = new BucketContext(bucket);

var query = from route in context.Query<Route>()
            join airport in context.Query<Airport>()
            on route.DestinationAirport equals airport.Faa
            select new {airport.AirportName, route.Airline};

foreach (var doc in query)
{
    // do work
}
```

[ANSI nest operations](https://github.com/couchbaselabs/Linq2Couchbase/blob/master/docs/nest.md#ansi-nests) and [hash join hints](https://github.com/couchbaselabs/Linq2Couchbase/blob/master/docs/query-hints.md#hash-joins) are also supported!

## Unix Milliseconds Date/Time Attributes

Some developers prefer to store their date/time values as Unix milliseconds (milliseconds since the start of the Unix epoch), rather than storing ISO8601 strings. They result in smaller documents, and may offer a slight boost in deserialization performance.

In the past, Linq2Couchbase didn't handle these fields well. It was possible to apply a custom serializer to the property, and this would support reads and writes, but comparisons or date operations were unsupported. Now, applying `[JsonConverter(typeof(UnixMillisecondsConverter))]` to a property will also allow comparisons. [See the documentation](https://github.com/couchbaselabs/Linq2Couchbase/blob/master/docs/date-handling.md) for more information on date/time handling.

```cs
[DocumentTypeFilter("my-document-type")]
public class MyDocument
{
    public long Id { get; set; }

    [JsonConverter(typeof(UnixMillisecondsConverter))]
    public DateTime Created { get; set; }
}

var query = from p in context.Query<MyDocument>()
    where p.Created < DateTime.Now
    select p;
```

## Enumerations Serialized as Strings

In the past, using `[JsonConverter(typeof(StringEnumConverter))]` would work for comparison operations (>, <, ==, !=, etc), but only if applied the enumeration definition itself. Now, the attribute may be applied to properties on the POCO.

```cs
[DocumentTypeFilter("my-document-type")]
public class MyDocument
{
    public long Id { get; set; }

    [JsonConverter(typeof(StringEnumConverter))]
    public ActionType ActionType { get; set; }
}

var query = from p in context.Query<MyDocument>()
    where p.ActionType == ActionType.Start
    select p;
```

## Other Custom Serializers

The new features around date/times and enumerations have been implemented using an extensible framework. It is also possible to add support for other custom serializers using [serialization converters](https://github.com/couchbaselabs/Linq2Couchbase/blob/master/docs/serialization-converters.md). This is also the approach for supporting the features above if you are not using Newtonsoft.Json and have some other method of applying custom serialization.

## Dictionary Operations

This improvement was actually included in 1.3.4, but was unannounced so I include it here. Dictionary operations can now be used within LINQ queries, which now recognize how the dictionary is serialized as a JSON object map. Supported operations include:

- Index accessor: `dictionary["key"]`
- Checking for key presense: `dictionary.ContainsKey("key")`
- Enumerating keys: `dictionary.Keys`
- Enumerating values: `dictionary.Values`

## Other Improvements

- Upgrade to Couchbase SDK 2.6.0
- Improvements to query generation performance [#241](https://github.com/couchbaselabs/Linq2Couchbase/pull/241)
