---
id: 68
title: jQuery.Net
date: 2009-05-02T10:49:25+00:00
guid: http://btburnett.com/?p=68
permalink: /2009/05/jquerynet.html
categories:
  - ASP.NET
  - jQuery
---
In the last few months, I've discovered [jQuery](http://jquery.com) and [jQuery UI](http://ui.jquery.com), which are a pair pretty awesome of open source javascript libraries.  I know, I'm a little behind the times here, but all I can say is "Wow".  Frankly, using jQuery makes the AJAX library that comes with .NET 3.0/3.5 from Microsoft seem foolish and clunky.  I've now transitioned away from using Microsoft's open source AJAX Control Library, instead choosing to use jQuery.

The only downside of using jQuery with ASP.Net is the lack of integration.  They now include a vsdoc file which helps with Intellisense, but there's a long way to go.  For simplicity, I'm still using Microsoft's AJAX code for their page methods and UpdatePanels, and primarily using jQuery for the client side controls and animations.

I've written a library to help with the creation of ASP.Net server controls that utilize jQuery.  I'm calling it, very creatively, jQuery.Net.  You can access the code and binaries for the library on github, the link's below.

<a href="http://github.com/btburnett3/jquery.net/tree/master" target="_blank">jQuery.Net Git Repository</a>
