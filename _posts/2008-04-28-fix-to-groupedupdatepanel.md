---
id: 9
title: Fix To GroupedUpdatePanel
date: 2008-04-28T10:59:00+00:00
guid: http://www.btburnett.com/?p=9
permalink: /2008/04/fix-to-groupedupdatepanel.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/04/fix-to-groupedupdatepanel.html
categories:
  - AJAX
  - ASP.NET
---
<div>
  <small>After further testing, I discovered a bug in my GroupedUpdatePanel. It didn&#8217;t always function correctly when the postback was initiated by a child control with ChildrenAsTriggers set to true. Any UpdatePanels declared after the initiated panel would update, but those before would not. I&#8217;ve corrected this, though it&#8217;s now a bit more complicated. There is an processing module which needs to be added to your web.config file in the section. Here&#8217;s the link to the corrected file.</small></p>

  <p>
    <small><a href="http://btburnett.com/downloads/GroupedUpdatePanel.zip">GroupedUpdatePanel.zip</a><br /> </small></div>
