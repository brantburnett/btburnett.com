---
id: 23
title: Extracting The Date From a SQL DateTime In SQL Queries
date: 2008-09-30T06:06:00+00:00
guid: http://www.btburnett.com/?p=23
permalink: /2008/09/extracting-the-date-from-a-sql-datetime-in-sql-queries.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/09/extracting-date-from-sql-datetime-in.html
categories:
  - SQL Server
---
<div xmlns="http://www.w3.org/1999/xhtml">
  <small>Ever wondered how to quickly and easily get the date portion of a datetime field in a SQL query? Seems like it should be easy, right? Unfortunately, it isn&#8217;t. I&#8217;ve found a lot of solutions, but this one is the most elegant I&#8217;ve seen.<br /></small></p>

  <pre class="csharpcode"><span class="kwrd">CONVERT</span>(datetime, FLOOR(<span class="kwrd">CONVERT</span>(<span class="kwrd">float</span>, DateTime)))</pre>

  <p>
    <small>I&#8217;ve put this into a user-defined function below:<br /></small>
  </p>

  <pre class="csharpcode"><span class="kwrd">SET</span> ANSI_NULLS <span class="kwrd">ON</span><br /><span class="kwrd">GO</span><br /><span class="kwrd">SET</span> QUOTED_IDENTIFIER <span class="kwrd">ON</span><br /><span class="kwrd">GO</span><br /><span class="kwrd">CREATE</span> <span class="kwrd">FUNCTION</span> [dbo].[ExtractDate]<br />(<br />   @DateTime datetime<br />)<br /><span class="kwrd">RETURNS</span> datetime<br /><span class="kwrd">AS</span><br /><span class="kwrd">BEGIN</span><br />   <span class="kwrd">RETURN</span> <span class="kwrd">CONVERT</span>(datetime, FLOOR(<span class="kwrd">CONVERT</span>(<span class="kwrd">float</span>, @DateTime)))<br /><span class="kwrd">END</span><br /></pre>

  <p>
    <small>This can be called like so:<br /></small>
  </p>

  <pre class="csharpcode"><span class="kwrd">SELECT</span> dbo.ExtractDate(DateTime) <span class="kwrd">FROM</span> Table</pre>

  <p>
    <small>Credit for this conversion method goes to <a href="http://mattberseth.com/">Matt Berseth</a>, i stumbled across it in the example code for something else.<br /></small></div>
