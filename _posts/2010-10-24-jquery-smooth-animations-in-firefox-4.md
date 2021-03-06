---
id: 148
title: jQuery Smooth Animations In Firefox 4
date: 2010-10-24T14:46:23+00:00
guid: http://btburnett.com/?p=148
permalink: /2010/10/jquery-smooth-animations-in-firefox-4.html
categories:
  - jQuery
---
[jQuery Smooth Animations](http://github.com/btburnett3/jquery.smoothanim) is an experiment in utilizing the new, [proposed animation API found in Firefox 4](http://weblogs.mozillazine.org/roc/archives/2010/08/mozrequestanima.html). It should be noted that this API is just a proposal and is experimental. Using it on live websites isn't recommended until the API is finalized.

jQuery Smooth Animations works by replacing parts of the standard jQuery API to make use of the new animation API if it is available. It should still be fully compatible with browsers that don't implement the animation API.

The end result of this plugin is that animations run more smoothly on Firefox 4, by syncing the animations with paint events. In my experimental comparisons between Firefox 4 Beta 6 and Firefox 3.6.12, I noticed a significant improvement in the jerkiness of the animation in the example.

I recognize that this plugin could be slightly less invasive by not replacing any of the API if window.mozRequestAnimationFrame is not found. However, I wanted to demonstrate how the code could function within the jQuery library itself rather than as a plugin. This is based on the theory that at some point, once the API is standardized, support will be integrated into the jQuery library.

Integration into the library would also allow us to eliminate the override of jQuery.now, instead using smarter logic within jQuery.fx.step. I simply didn't want to replace jQuery.fx.step because of the extensive logic it contains which could be changed in future releases of jQuery.

[GitHub Repository](http://github.com/btburnett3/jquery.smoothanim)
