---
id: 16
title: Easily Work With Differing Time Zones In An ASP.NET 3.5 Application
date: 2008-06-18T05:36:00+00:00
guid: http://www.btburnett.com/?p=16
permalink: /2008/06/easily-work-with-differing-time-zones-in-an-aspnet-35-application.html
categories:
  - ASP.NET
---
When working with ASP.NET applications, time zones are often a problem when dealing with DateTime structures. There are two different common scenarios that a developer is likely to encounter. The first is that you are placing the application on a hosted server that is in a different time zone than the business you are trying to operate. The second is that you have users from different time zones and would like to display the date and time to the user in their local time zone rather than the server's time zone.

Hosted Time Zone addresses this problem by providing two settings. ApplicationTimeZone sets the time zone to be used by the entire application. ThreadTimeZone sets the time zone to be used by the currently running thread, which defaults to the ApplicationTimeZone if no time zone has been specified on the thread. If you are using ThreadTimeZone then you need to be sure to set the value for each request made to the server. You will usually do this in your global.asax file, such as in the Application_BeginRequest handler.

To actually perform the conversions you can call the ToAppTime, FromAppTime, ToThreadTime, or FromThreadTime static methods on the ApplicationTimeZone class. Additionally, extension methods have been declared for the DateTime and DateTimeOffset classes that perform the same functions more cleanly. In order for the extension methods to work you must import the HostedTimeZone namespace.

You should note that there are not any overloads for FromAppTime or FromThreadTime for the DateTimeOffset class. This is because the ToLocalTime method of DateTimeOffset performs the same function, so they would be redundant.

This library requires the TimeZoneInfo class, which is not available until the .NET Framework 3.5. Therefore, it is not compatible with ASP.NET 2.0 or earlier.

[Hosted Time Zone 1.0.0.0](/downloads/HostedTimeZone.1.0.0.0.zip)
