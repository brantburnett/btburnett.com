---
id: 112
title: jQuery.themescript 1.1.0 Released
date: 2010-02-09T18:51:39+00:00
guid: http://btburnett.com/?p=112
permalink: /2010/02/jquery-themescript-1-1-0-released.html
categories:
  - jQuery
---
Today I've released version 1.1.0 of jQuery.themescript.  This release includes some significant improvements that make it far more functional in the real world.

The original design concept was for a web store framework I was working on.  I wanted each store be able to easily change their javascript based themes without making changes to the javascript files for the basic theme.  This made for a much easier file based deployment of new versions to each store.  The obvious downside is that you're downloading lots of extra javascript, particularly if you're turning features back off that were in the basic theme.  This type of functionality is only useful in certain cases, and is never the most efficient system possible with regards to performance and bandwidth, just maintainability.

This new release still maintains that basic functionality, but now I address a more common concern as well, applying themes to AJAX updates.  In order to support this, I've added several new features.

* $.themescript.exec now accepts a context parameter.  This can be a jQuery object or an HTML DOM element, just like the context to jQuery's [jQuery(selector, [context])](http://api.jquery.com/jQuery/) selector function.  All registered functions will receive this context as a parameter, and it will be automatically used as the context to selector-based theming to restrict the elements returned.
* themescript is a new function added to jQuery objects which allows you to execute the themescripts against a specific jQuery object.  $('#selector').themescript() is equivalent to $.themescript.exec( $('#selector') )
* themescript( url, [data], [callback] ) works just like the jQuery [load](http://api.jquery.com/load/) method, except that it will automatically run themescripts against the updated HTML if the request is successful.

You can also easily add support for jQuery.themescript to ASP.Net AJAX partial page updates (a.k.a. UpdatePanels).  Just add this script code to your files and the themescripts will be automatically executed against any UpdatePanels which are changed during an asynchronous postback.

```js
var prm = Sys.WebForms.PageRequestManager.getInstance();
prm.add_pageLoaded(function(sender, args) {
    var&lt;/span&gt; updated = args.get_panelsUpdated();
    if&lt;/span&gt; (updated && updated.length) {
        $.themescript.exec($(updated));
    }
});
```

Here are the links where you can access the GitHub repository or download the files:

[GitHub](http://github.com/btburnett3/jquery.themescript)

[Download](http://github.com/btburnett3/jquery.themescript/zipball/1.1.0)
