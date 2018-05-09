---
id: 248
title: Cancelling Long Running Couchbase N1QL Queries
date: 2017-02-19T12:23:36+00:00
guid: http://btburnett.com/?p=248
permalink: /2017/02/cancelling-long-running-couchbase-n1ql-queries.html
categories:
  - Couchbase
  - MVC
---
The recent release of the [Couchbase .NET SDK 2.4.0](https://blog.couchbase.com/introducing-couchbase-net-2-4-0-net-core-ga/) has added many new features.  There is a minor feature, however, that is worth a mention.  It&#8217;s now possible to cancel long-running N1QL queries.

For example, in a web application a user might browse away from the page in impatience.  When they do, you don&#8217;t want  the query to keep executing pointlessly.  Instead, you can cancel the query, freeing web server and Couchbase  server resources for other requests.

## How To Cancel

Using this new feature is very easy, when executing your query simply supply a CancellationToken.  For web applications, this can be acquired by including a CancellationToken as a parameter on an asynchronous action method.

```cs
public async Task&lt;ActionResult&gt; Index(CancellationToken cancellationToken)
{
    var bucket = ClusterHelper.GetBucket("my-bucket");
    var query = new QueryRequest("SELECT * FROM `my-bucket` WHERE type = 'docType' LIMIT 1000");

    var result = await bucket.QueryAsync&lt;Document&gt;(query, cancellationToken);
    if (result.Success) {
        return Json(result.Rows);
    }
}
```

## Compatibility

Note: Documentation on what versions of ASP.NET MVC and Web API support CancellationToken is a bit sparse.  Apparently some versions only use it for timeouts (using  AsyncTimeout), while some versions have support for cancellations from the browser.  There is also a [way to add support for browser cancellation using CreateLinkedTokenSource](http://dirk.schuermans.me/?p=749).

The behavior may also depend on the web server you&#8217;re using (i.e. IIS versus Kestrel, and Kestrel version). For example, [this change to Kestrel](https://github.com/aspnet/KestrelHttpServer/pull/1218) appears to cause client disconnects to do a better job of triggering the CancellationTokken. If anyone knows more details about version support, please let me know and I&#8217;ll update the post.