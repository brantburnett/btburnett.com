---
id: 179
title: Query NoSQL From .Net Using Couchbase and LINQ
date: 2015-10-27T17:20:15+00:00
guid: http://btburnett.com/?p=179
permalink: /2015/10/query-nosql-from-net-using-couchbase-and-linq.html
categories:
  - Couchbase
---
Historically, one of the biggest hurdles dealing with NoSQL solutions has been querying the data.  In particular, the learning curve for new developers trained in SQL has been very steep.

Couchbase recently helped to address this limitation with the release of N1QL (a SQL variant) in [Couchbase Server 4.0](http://blog.couchbase.com/2015/october/announcing-couchbase-server-4.0).  N1QL vastly expands upon both the power of Couchbase as well as the ease with which developers can query the data.  Because the language is so similar to SQL, developers can quickly make the transition.  As an example, at [CenterEdge](http://centeredgesoftware.com/) we recently hired a new developer, and we had him up and working with N1QL queries within a day.

Now, Couchbase has [announced the release of their Couchbase LINQ Provider](http://blog.couchbase.com/2015/october/linq-n1ql-and-couchbase-oh-mai-linq2couchbase-ga) for .Net.  The provider is [available as a Nuget package](https://www.nuget.org/packages/Linq2Couchbase/) which adds support to the existing SDK.

For .Net developers, using the LINQ provider for Couchbase makes the learning curve even shallower.  With just a few exceptions, developers trained to write queries using LINQ with NHibernate or Entity Framework can make the transition without learning any new syntax.  Plus, due to Couchbase&#8217;s JSON document data structure, the LINQ provider adds support for a variety of N1QL features that aren&#8217;t possible with more traditional table-based data stores.

So, if you&#8217;re a .Net developer interested in NoSQL and big data, I strongly encourage you to check it out.  It&#8217;s a big step in bringing the performance and scalability of NoSQL to the masses without the headaches.

Disclaimer:  I&#8217;m a extra excited about this release because it is an open source project I was able to contribute to.  You can [see the source code on GitHub](https://github.com/couchbaselabs/Linq2Couchbase).