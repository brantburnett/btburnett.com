---
id: 15
title: Using URL Rewriting With Web Parts
date: 2008-06-10T12:00:00+00:00
guid: http://www.btburnett.com/?p=15
permalink: /2008/06/using-url-rewriting-with-web-parts.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/06/using-url-rewriting-with-web-parts.html
categories:
  - ASP.NET
  - URL rewriting
  - Web Parts
---
<div>
  <small>Web parts are a great new system for allowing personalization of web sites by end users that<br /> was introduced in ASP.NET 2.0. However, it isn&#8217;t useful when used in combination with<br /> URL rewriting.</p>

  <p>
    URL rewriting allows you to respond to requests from dynamic URLs by redirecting the handling<br /> to a different .ASPX file on your server. This can be used to make your URLs more readable<br /> and improve your search engine optimization (SEO), instead of using long query parameters.<br /> Some example systems for implementing this are UrlRewriter.net and UrlRewriting.net.
  </p>

  <p>
    When you use web parts in combination with a rewritten URL, the web parts will all be shared by<br /> all of the URLs that are rewritten to the same .ASPX file. This library corrects this<br /> by replacing the WebPartManager with RewritableWebPartManager and the SqlPersonalizationProvider<br /> with RewritableSqlPersonalizationProvider.
  </p>

  <p>
    It also provides additional functionality in the form a personalization levels. This allows you<br /> to share common personalization settings amonst multiple URLs. When a user is browsing, if no<br /> personalization is specified at a particular level, the personalization defined at a higher level<br /> is displayed instead. For example, in a web store scenario you can allow the store operators to<br /> provide shared personalization that all users will see. Using levels, they can set personalization<br /> that will appear on all item pages, such as a disclaimer, and then override that personalization<br /> with a different disclaimer on a specific item&#8217;s page.
  </p>

  <p>
    More information can be found in the readme file included in the download.
  </p>

  <p>
    <big><a href="http://btburnett.com/downloads/RewritableWebPartsLibrary.1.0.0.0.zip">Rewritable Web Parts Library 1.0.0.0</a></big><br /> </small></div>
