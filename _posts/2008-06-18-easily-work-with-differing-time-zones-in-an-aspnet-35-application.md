---
id: 16
title: Easily Work With Differing Time Zones In An ASP.NET 3.5 Application
date: 2008-06-18T05:36:00+00:00
guid: http://www.btburnett.com/?p=16
permalink: /2008/06/easily-work-with-differing-time-zones-in-an-aspnet-35-application.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/06/easily-work-with-differing-time-zones.html
categories:
  - ASP.NET
---
<div>
  <small>When working with ASP.NET applications, time zones are often a problem when dealing with<br /> DateTime structures. There are two different common scenarios that a developer is likely<br /> to encounter. The first is that you are placing the application on a hosted server<br /> that is in a different time zone than the business you are trying to operate. The second<br /> is that you have users from different time zones and would like to display the date and time<br /> to the user in their local time zone rather than the server&#8217;s time zone.</p>

  <p>
    Hosted Time Zone addresses this problem by providing two settings. ApplicationTimeZone sets<br /> the time zone to be used by the entire application. ThreadTimeZone sets the time zone to be<br /> used by the currently running thread, which defaults to the ApplicationTimeZone if no time<br /> zone has been specified on the thread. If you are using ThreadTimeZone then you need to be<br /> sure to set the value for each request made to the server. You will usually do this in your<br /> global.asax file, such as in the Application_BeginRequest handler.
  </p>

  <p>
    To actually perform the conversions you can call the ToAppTime, FromAppTime, ToThreadTime, or<br /> FromThreadTime static methods on the ApplicationTimeZone class. Additionally, extension methods<br /> have been declared for the DateTime and DateTimeOffset classes that perform the same functions<br /> more cleanly. In order for the extension methods to work you must import the HostedTimeZone<br /> namespace.
  </p>

  <p>
    You should note that there are not any overloads for FromAppTime or FromThreadTime for the<br /> DateTimeOffset class. This is because the ToLocalTime method of DateTimeOffset performs the<br /> same function, so they would be redundant.
  </p>

  <p>
    This library requires the TimeZoneInfo class, which is not available until the .NET Framework 3.5.<br /> Therefore, it is not compatible with ASP.NET 2.0 or earlier.
  </p>

  <p>
    <big><a href="http://btburnett.com/downloads/HostedTimeZone.1.0.0.0.zip">Hosted Time Zone 1.0.0.0</a></big><br /> </small></div>
