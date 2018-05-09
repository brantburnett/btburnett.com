---
id: 100
title: Simplify Asynchronous Javascript Loading In ASP.Net Using LABjs
date: 2010-01-27T17:41:48+00:00
guid: http://btburnett.com/?p=100
permalink: /2010/01/simplify-asynchronous-javascript-loading-in-asp-net.html
categories:
  - ASP.NET
  - Javascript
  - jQuery
---
[LABjs](http://labjs.com) is an excellent javascript library that performs asynchronous loading of javascript files. This can help to greatly increase the load speed of your web pages. Now, instead of blocking while one file is being downloaded, other scripts further down the chain can be downloaded while waiting. On top of that, it can maintain processing order, waiting to process certain scripts until others are complete, and optionally executing code on completion.

All of this is great for the client side. But how do you define which scripts to include in the LABjs chain on the server side?

Since for work I primarily operate in ASP.Net, I&#8217;ve created an ASP.Net solution to the problem, LABjs.Net. This library provides two key controls, LabScriptManager and LabScriptManagerProxy. These controls loosely follow the behavior of the AJAX ScriptManager and ScriptManagerProxy controls, at least their script loading aspects.

Key supported features include:

1. Refer to script files using application relative paths (i.e. ~/js/jquery.min.js)
2. Load script files from assembly resources
3. Specify if debug or release scripts should be used, or use the debug setting from the web.config file
4. Ability to set any of the options provided by LABjs
5. Include wait() calls in the chain, and provide inline functions to be executed after the wait
6. Use LabScriptManagerProxy to add scripts and waits to the chain in content pages and user controls
7. Use LabActionGroup inside LabScriptManagerProxy to add script() calls at a specific point in the primary chain
8. LABjs debug and release versions are embedded in the DLL and automatically referenced, but you can opti0nally override this with your own URL
9. Experimental support for the cdnLABjs library I am working on, which provides automatic failover to a local file if a file fails to load from a CDN (for information about why, see [Using CDN Hosted jQuery with a Local Fall-back Copy](http://bit.ly/8YKQ2f))

Below you will find links to the current release candidate, 1.0rc1.  Please review it and give me any feedback you might have.

[Download Binaries](http://cloud.github.com/downloads/btburnett3/LABjs.Net/LABjs.Net-1.0rc3.zip)

[Readme File](http://github.com/btburnett3/LABjs.Net/raw/master/README)

[Git Repository](http://github.com/btburnett3/LABjs.Net)

**Update 2/8/2010:** [Updated to version 1.0rc4](http://btburnett.com/2010/02/labjs-net-release-candidate-1-0rc4.html)