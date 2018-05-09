---
id: 15
title: Using URL Rewriting With Web Parts
date: 2008-06-10T12:00:00+00:00
guid: http://www.btburnett.com/?p=15
permalink: /2008/06/using-url-rewriting-with-web-parts.html
categories:
  - ASP.NET
---
Web parts are a great new system for allowing personalization of web sites by end users that was introduced in ASP.NET 2.0. However, it isn't useful when used in combination with URL rewriting.

URL rewriting allows you to respond to requests from dynamic URLs by redirecting the handling to a different .ASPX file on your server. This can be used to make your URLs more readable and improve your search engine optimization (SEO), instead of using long query parameters. Some example systems for implementing this are UrlRewriter.net and UrlRewriting.net.

When you use web parts in combination with a rewritten URL, the web parts will all be shared by all of the URLs that are rewritten to the same .ASPX file. This library corrects this by replacing the WebPartManager with RewritableWebPartManager and the SqlPersonalizationProvider with RewritableSqlPersonalizationProvider.

It also provides additional functionality in the form a personalization levels. This allows you to share common personalization settings amonst multiple URLs. When a user is browsing, if no personalization is specified at a particular level, the personalization defined at a higher level is displayed instead. For example, in a web store scenario you can allow the store operators to provide shared personalization that all users will see. Using levels, they can set personalization that will appear on all item pages, such as a disclaimer, and then override that personalization with a different disclaimer on a specific item's page.

More information can be found in the readme file included in the download.

[Rewritable Web Parts Library 1.0.0.0](/downloads/RewritableWebPartsLibrary.1.0.0.0.zip)
