---
id: 14
title: Forcing 32-bit Execution in .NET
date: 2008-06-03T05:55:00+00:00
guid: http://www.btburnett.com/?p=14
permalink: /2008/06/forcing-32-bit-execution-in-net.html
categories:
  - .NET Framework
---
Sometimes when developing .NET applications it becomes necessary to force an application to run in 32-bit mode, even on a 64-bit processor. One scenario that I've run into is when you're using Crystal Reports embedded in the application. I'm not sure about newer versions, but for Crystal Reports XI R2 it won't work in 64-bit mode.

In order to get something like that to work, you must force 32-bit execution on the executable. Forcing it on a DLL assembly in your application won't help, then it won't be able to load that DLL either. It needs to start in 32-bit mode from the beginning. If you can make the change in the development environment before compilation, just set the flags on the assembly there and everything will be great.

However, if you need to make the change to a compiled assembly it's a little more difficult. To do so, use the corflags command-line utility. This program is included in the Windows SDK. To do so, simply run `corflags program.exe /32BIT+`.

If the assembly is strongly-named, then you must do a little more. First, you must add a /force flag, running `corflags program.exe /32BIT+ /Force`. Then, you must rehash and resign the assembly, using "sn -Ra program.exe key.snk". In this case, key.snk is your key file for signing the assembly.

Hope this helps!
