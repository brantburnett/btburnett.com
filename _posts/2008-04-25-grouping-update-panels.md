---
id: 8
title: Grouping Update Panels
date: 2008-04-25T06:19:00+00:00
guid: http://www.btburnett.com/?p=8
permalink: /2008/04/grouping-update-panels.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/04/grouping-update-panels.html
categories:
  - AJAX
  - ASP.NET
---
<div>
  <small>Without a doubt, AJAX is one of the most useful web development tools out there. Through AJAX, we finally have a method for developing web pages with robust user interfaces.</small></p>

  <p>
    <small>The most common and simplest method of AJAX development on ASP.NET is to use the UpdatePanel. While the UpdatePanel creates a lot of overhead, it is very simple to implement. However, sometimes you will have multiple UpdatePanels on a page that relate to each other. In order to reduce your overhead, you will probably want to set thier UpdateMode to Conditional. With the Conditional UpdateMode enabled, they will only update under three conditions (note that nested UpdatePanels are a bit more complicated):<small><br /> </small></small>
  </p>

  <ol>
    <li>
      <small>A child control of the UpdatePanel performs a postback, and ChildrenAsTriggers is true</small>
    </li>
    <li>
      <small>The Update method is explicitly called.</small>
    </li>
    <li>
      <small>A manually defined trigger is fired.</small>
    </li>
  </ol>

  <p>
    <small>However, this doesn&#8217;t help with firing the other UpdatePanels on the page that are related. While you can update the other panels manually using the Update method or triggers, that can potentially be a lot of work and difficult to maintain.</small>
  </p>

  <p>
    <small>In order to address this, I&#8217;ve created a simple GroupedUpdatePanel. It has a GroupName property which is used to group multiple GroupedUpdatePanels together. If any one UpdatePanel in a group is updated as part of an asynchronous postback, they will all be updated.</small>
  </p>

  <p>
    <small>There are two important restrictions to note. First, you must set UpdateMode to Conditional. If any one GroupedUpdatePanel in a group is set to Always, they will all update every time. This is a result of the limitations of inheriting from the UpdatePanel class. Second, all of the UpdatePanels must be created and on the page at the time the PreRender event occurs. I&#8217;ve seen some reports on the web of certain situations where this is not the case, though I haven&#8217;t tested for any of it.</small>
  </p>

  <p>
    <small>You can download the source for the control here. Feel free to use it in your own applications.</small>
  </p>

  <p>
    <strong>UPDATE: Please see updated version <a href="http://btburnett.com/2008/04/fix-to-groupedupdatepanelfix-to-groupedupdatepanel.html">here</a></strong></div>
