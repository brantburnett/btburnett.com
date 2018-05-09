---
id: 152
title: MVC 3 Unobtrusive AJAX Improvements
date: 2011-01-28T18:34:13+00:00
guid: http://btburnett.com/?p=152
permalink: /2011/01/mvc-3-unobtrusive-ajax-improvements.html
categories:
  - MVC
---
I started experimenting this week with MVC 3 and the new unobtrustive javascript frameworks, both for AJAX and validation. These are built around jQuery, and frankly are very nice pieces of work. There are, however, a few improvements that can be made to the Javascript files included in the distribution. You can obviously include these changes manually, it's pretty simple, but hopefully one day Microsoft will roll these up in the next version.

I'm not sure how the unobtrusive stuff is licensed by Microsoft, so I won't publish the modified code here until I'm certain it's okay. I will, however, go over the modifications so that you can do them yourselves.

1. **Use fn.delegate instead of fn.live &#8211;** By default, the unobtrusive AJAX library uses fn.live to monitor for anchor click and form submit events, and if the elements are set to operate using AJAX they are intercepted and an AJAX request is initiated.  However, this can be inefficient because the selector is actually being fired during during document.ready, even though it isn't used for anything.  This means every single form, submit button, and link on your page is being checked for a data-ajax="true" attribute during document.ready for no reason.  By using $(document).delegate(_selector_, _eventType_, _fn_) this is completely avoided. _**Note**: This only works using jQuery 1.4.2 or later._
2. **Cache jQuery Objects** &#8211; While this is a minor efficiency concern, there are several places with the event handlers where $(form) is called multiple times to convert an element into a jQuery object with a single member.  It is much more efficient to cache $(form) in a local variable and reuse it. You can also make finding the form more efficient using `.closest("form")` instead of `.parents("form")[0]`.
3. **Inefficient Handling Of Null Callbacks** &#8211; asyncRequest uses getFunction to get a callback function based on data attributes.  This is nifty system, because can take either a function name or inline javascript code.  However, if there is no function specified, it goes through the trouble of using Function.constructor to create an empty function.  This can be avoided by adding "if (!code) return $.noop;" to the beginning of getFunction.
4. **Global AJAX Events** &#8211; Currently, there is no way to attach global AJAX event handlers that specifically handle only the MVC related AJAX events.  Your only options are to intercept ALL jQuery AJAX events, or to attach events to each specific AJAX request using data attributes.  However, there are definitely use cases for global event handlers that only affect MVC related AJAX events.  Therefore, I recommend updating the "asyncRequest" function to add information to the options passed to $.ajax.  In particular, during the $.extend(options, {&#8230;}) I am adding "mvcRequest: true" and "mvcTargetElement: element".  Now you can attach global jQuery AJAX handlers using $(document).ajaxSuccess, etc, and then test to see if it is an MVC request.
5. **AJAX Updated Forms Validation** &#8211; In the current implementation of the unobtrusive validation, it can't handle forms added to the page via an AJAX request.  You have to manually call $.validate.unobtrusive.parse() after the page update before client-side validation resumes.  This can be addressed using a global handler, so long as #3 above is also implemented.  By using the attributes passed in, we restrict the processing so it doesn't re-parse the entire document, only the updated region. The code for this is below:

```js
$(document).ajaxSuccess(function (event, xhr, settings) {
    if (settings.mvcTargetElement) {
        $(settings.mvcTargetElement.getAttribute("data-ajax-update")).each(function () {
            $.validator.unobtrusive.parse(this);
        });
    }
});
```

Please let me know if you see any other improvements that I've missed. I'm just beginning to dig through things, so I'll keep updating as I find more as well.
