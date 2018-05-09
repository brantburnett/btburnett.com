---
id: 22
title: SQL Server 2005 Service Broker Error Code 25 (Event ID 28054)
date: 2008-09-02T12:06:00+00:00
guid: http://www.btburnett.com/?p=22
permalink: /2008/09/sql-server-2005-service-broker-error-code-25-event-id-28054.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/09/sql-server-2005-service-broker-error.html
categories:
  - SQL Server
---
<div xmlns="http://www.w3.org/1999/xhtml">
  <span id="_ctl0_MainContent_PostFlatView"><span><small>Another issue I&#8217;ve run into with the service broker and moving databases between servers is this error message:<br /></small></span></span></p>

  <blockquote>
    <p>
      <span id="_ctl0_MainContent_PostFlatView"><span><small>Error code:25. The master key has to exist and the service master key encryption is required.</small></span></span><br /><span id="_ctl0_MainContent_PostFlatView"><span></span></span>
    </p>
  </blockquote>

  <p>
    <span id="_ctl0_MainContent_PostFlatView"><span><small>The solution that I found was the regenerate the key for the database using the following code:<br /></small></span></span>
  </p>

  <blockquote>
    <p>
      <span id="_ctl0_MainContent_PostFlatView"><span><small>CREATE MASTER KEY ENCRYPTION BY PASSWORD = &#8216;1ComplexPassword!&#8217;</small></span></span><br /><span id="_ctl0_MainContent_PostFlatView"><span></span></span>
    </p>
  </blockquote>

  <p>
    <span id="_ctl0_MainContent_PostFlatView"><span><small>You might also need to run the following afterwards:<br /></small></span></span>
  </p>

  <blockquote>
    <p>
      <span id="_ctl0_MainContent_PostFlatView"><span><small>ALTER DATABASE dbname SET NEW_BROKER.</small></span></span><br /><span id="_ctl0_MainContent_PostFlatView"><span></span></span>
    </p>
  </blockquote>

  <p>
    <span id="_ctl0_MainContent_PostFlatView"><span><small>Also, see my previous post <a href="http://blog.btburnett.com/2008/05/sql-server-2005-service-broker-error.html">SQL Server 2005 Service Broker Error 15517</a> for another error message and solution.</small></span><br /> </span></div>
