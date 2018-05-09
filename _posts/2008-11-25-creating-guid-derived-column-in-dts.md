---
id: 25
title: Creating GUID Derived Column in DTS
date: 2008-11-25T11:32:00+00:00
guid: http://www.btburnett.com/?p=25
permalink: /2008/11/creating-guid-derived-column-in-dts.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/11/creating-guid-derived-column-in-dts.html
categories:
  - SQL Server
  - SSIS/DTS
---
If you want to add a constant GUID to a table during a DTS transformation, it&#8217;s a little tricky.  Here&#8217;s the solution.

Use the Derived Column transformation in order to add a new column.  For the formula, use the GUID surrounded by double quotes.  The trick is that you MUST include the curly braces, like so:

```cs
"{B83030CF-EC5A-45ca-BE2D-BCFCC2A85034}"
```

Then set the type of the column to unique identifier (DT_GUID).  That&#8217;s it.