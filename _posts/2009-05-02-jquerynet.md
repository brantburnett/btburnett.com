---
id: 68
title: jQuery.Net
date: 2009-05-02T10:49:25+00:00
guid: http://btburnett.com/?p=68
permalink: /2009/05/jquerynet.html
categories:
  - AJAX
  - ASP.NET
  - jQuery
---
In the last few months, I&#8217;ve discovered [jQuery](http://jquery.com) and [jQuery UI](http://ui.jquery.com), which are a pair pretty awesome of open source javascript libraries.  I know, I&#8217;m a little behind the times here, but all I can say is &#8220;Wow&#8221;.  Frankly, using jQuery makes the AJAX library that comes with .NET 3.0/3.5 from Microsoft seem foolish and clunky.  I&#8217;ve now transitioned away from using Microsoft&#8217;s open source AJAX Control Library, instead choosing to use jQuery.

The only downside of using jQuery with ASP.Net is the lack of integration.  They now include a vsdoc file which helps with Intellisense, but there&#8217;s a long way to go.  For simplicity, I&#8217;m still using Microsoft&#8217;s AJAX code for their page methods and UpdatePanels, and primarily using jQuery for the client side controls and animations.

I&#8217;ve written a library to help with the creation of ASP.Net server controls that utilize jQuery.  I&#8217;m calling it, very creatively, jQuery.Net.  You can access the code and binaries for the library on github, the link&#8217;s below.

<a href="http://github.com/btburnett3/jquery.net/tree/master" target="_blank">jQuery.Net Git Repository</a>