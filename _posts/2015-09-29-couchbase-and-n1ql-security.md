---
id: 174
title: Couchbase and N1QL Security
date: 2015-09-29T19:58:38+00:00
guid: http://btburnett.com/?p=174
permalink: /2015/09/couchbase-and-n1ql-security.html
categories:
  - Couchbase
---
As a developer at [CenterEdge Software](http://centeredgesoftware.com/), I&#8217;ve had a lot of cause to use [Couchbase](http://www.couchbase.com/) as our NoSQL databasing platform over the last few years.  I&#8217;ve gotten really excited about the potential of the new Couchbase query language in Couchbase 4.0, called N1QL.  So excited that I&#8217;ve spent a lot of time contributing to the [Linq2Couchbase](https://github.com/couchbaselabs/Linq2Couchbase) library, which allows developers to use LINQ to transparently create N1QL queries.

In doing work with N1QL, I quickly realized that it may have some of the same security concerns as SQL.  In particular, N1QL injection could be a new surface area for attack in Couchbase 4.0.  That&#8217;s what I call the N1QL equivalent of SQL injection.  I found that while the risks are lower in N1QL than in SQL, there are still some areas that need to be addressed by application developers using Couchbase.

As a result, I did some research and recently wrote a guest post on N1QL security for Couchbase users.  It researches possible N1QL injection security concerns, then goes into how to protect your applications when using N1QL.

<http://blog.couchbase.com/2015/september/couchbase-and-n1ql-security-centeredgesoftware>