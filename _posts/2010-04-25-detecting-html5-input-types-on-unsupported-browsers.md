---
id: 132
title: Detecting Original HTML5 Input Types On Unsupported Browsers
date: 2010-04-25T23:24:43+00:00
guid: http://btburnett.com/?p=132
permalink: /2010/04/detecting-html5-input-types-on-unsupported-browsers.html
categories:
  - jQuery
---
I&#8217;ve been doing some work recently on extending [jQuery UI](http://jqueryui.com/) to style form elements using the [Themeroller](http://jqueryui.com/themeroller/).  One thing that I wanted to implement was detection of the new HTML5 input types and choosing the correct widget to use based upon that.  This would provide progressive enhancement of the input element to support the HTML5 input type, even if that type of input isn&#8217;t supported on that browser.

By definition, if a browser can&#8217;t support the type of an input, it automatically falls back to a text input.  This is great for backwards compatibility, but it gave me a bit of a problem.  If I set type=&#8221;number&#8221; in my HTML, testing the type of the element in Javascript returns &#8220;text&#8221; when the number element isn&#8217;t supported.  Useful for determining if support is available, not so useful for determining what the original value is in the HTML.

In most browsers, you can find the original value using the attributes collection.  However, this doesn&#8217;t work in IE7 and earlier.  And, as everyone knows, we still have to support IE 6 & 7 for most web sites.  I found an alternative for IE, which is to test the outerHTML of the element using a regular expression.  IE will return this with the original attributes intact.

I&#8217;ve written a jQuery filter extension that allows you to test any element for the original input type:

```js
// add HTML5 input type expression (still detects HTML5 input types on browsers that don't support them)
$.extend($.expr[':'], {
    inputtype: function(elem, i, type) {
        function getRawAttr() {
            // IE will return the original value in the outerHTML
            var match = /&lt;input.*?\s+type=(\w+)(\s+|&gt;).*?/i.exec(elem.outerHTML);

            if (match && match.length &gt;= 2 && match[1] !== "text") {
                return match[1];
            }

            // for other browsers, test attributes collection (doesn't work in IE&lt;7)
            var attrs = elem.attributes,
                i;

            for (i=0; i&lt;attrs.length; i++) {
                if (attrs[i].nodeName === "type") {
                    return attrs[i].nodeValue;
                }
            }

        }

        if (elem.tagName != "INPUT") return false;
        if (elem.type === "text") {
            // could be unsupported type fallback, so do further testing
            return getRawAttr() === type[3];
        } else {
            return elem.type === type[3];
        }
    }
});
```

To use this extension, just use the :inputtype(type) filter in your jQuery expression.  For example:

<pre class="brush: jscript; title: ; notranslate" title="">$('input:inputtype(number)).width(100);

if ($(elem).is(':inputtype(number)')) {
...
}

</pre>

**Update:** Changed to only run the check if the tag is an input tag, in case you run the filter against non-input elements.