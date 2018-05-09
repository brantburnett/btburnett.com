---
id: 25
title: Creating GUID Derived Column in DTS
date: 2008-11-25T11:32:00+00:00
guid: http://www.btburnett.com/?p=25
permalink: /2008/11/creating-guid-derived-column-in-dts.html
categories:
  - SQL Server
---
If you want to add a constant GUID to a table during a DTS transformation, it's a little tricky.  Here's the solution.

Use the Derived Column transformation in order to add a new column.  For the formula, use the GUID surrounded by double quotes.  The trick is that you MUST include the curly braces, like so:

```cs
"{B83030CF-EC5A-45ca-BE2D-BCFCC2A85034}"
```

Then set the type of the column to unique identifier (DT_GUID).  That's it.
