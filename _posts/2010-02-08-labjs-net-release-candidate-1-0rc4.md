---
id: 109
title: LABjs.Net Release Candidate 1.0rc4
date: 2010-02-08T11:18:27+00:00
guid: http://btburnett.com/?p=109
permalink: /2010/02/labjs-net-release-candidate-1-0rc4.html
categories:
  - ASP.Net
---
I've committed release candidate 4 of LABjs.Net at GitHub.  It contains a few minor bug fixes, and also adds a few useful features.

* LabScriptCombine allows you to make use of the automated runtime script combining features of ASP.Net 3.5 in combination with LABjs script chaining.  To get the optimum script load time, combine scripts between each wait() call into a single HTTP request, so long as you are consistently using that combination of scripts across your pages.
* Added some useful constructors to several of the LABjs.Net classes.  These constructors make it easier to build a LABjs chain in your code rather than in your .aspx files.
* Made a slight change to the LabWait constructor that accepts an inlineScript parameter.  It will now default DetectScriptTags to False if you use this constructor, since you shouldn't be including the script tags if you are building your chain in code.  You can still set this Property back to True if needed.
* Added AlternateRef property to LabScriptReference.  If you're using the experimental CDN failover extension to LABjs, it allows you to specify a full set of options for the alternate script.  The AlternateRef property allows you to make use of those options if you want to, by providing a full LabScriptReference class to use.  This is optionally used instead of the simpler Alternate property that just provides a URL.

[Binaries](http://cloud.github.com/downloads/btburnett3/LABjs.Net/LABjs.Net-1.0rc4.zip)
[GitHub](http://github.com/btburnett3/LABjs.Net)
