---
id: 14
title: Forcing 32-bit Execution in .NET
date: 2008-06-03T05:55:00+00:00
guid: http://www.btburnett.com/?p=14
permalink: /2008/06/forcing-32-bit-execution-in-net.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/06/forcing-32-bit-execution-in-net.html
categories:
  - .NET Framework
---
<div xmlns='http://www.w3.org/1999/xhtml'>
  <small>Sometimes when developing .NET applications it becomes necessary to force an application to run in 32-bit mode, even on a 64-bit processor. One scenario that I&#8217;ve run into is when you&#8217;re using Crystal Reports embedded in the application. I&#8217;m not sure about newer versions, but for Crystal Reports XI R2 it won&#8217;t work in 64-bit mode.</p>

  <p>
    In order to get something like that to work, you must force 32-bit execution on the executable. Forcing it on a DLL assembly in your application won&#8217;t help, then it won&#8217;t be able to load that DLL either. It needs to start in 32-bit mode from the beginning. If you can make the change in the development environment before compilation, just set the flags on the assembly there and everything will be great.
  </p>

  <p>
    However, if you need to make the change to a compiled assembly it&#8217;s a little more difficult. To do so, use the corflags command-line utility. This program is included in the Windows SDK. To do so, simply run &#8220;corflags program.exe /32BIT+&#8221;.
  </p>

  <p>
    If the assembly is strongly-named, then you must do a little more. First, you must add a /force flag, running &#8220;corflags program.exe /32BIT+ /Force&#8221;. Then, you must rehash and resign the assembly, using &#8220;sn -Ra program.exe key.snk&#8221;. In this case, key.snk is your key file for signing the assembly.
  </p>

  <p>
    Hope this helps!<br /></small></div>
