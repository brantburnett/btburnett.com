---
id: 26
title: Combining Date and Time in DTS To Make a DT_DBTIMESTAMP datetime
date: 2008-11-25T12:16:00+00:00
guid: http://www.btburnett.com/?p=26
permalink: /2008/11/combining-date-and-time-in-dts-to-make-a-dt_dbtimestamp-datetime.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/11/combining-date-and-time-in-dts-to-make.html
categories:
  - SQL Server
  - SSIS/DTS
---
I was recently dealing with importing a DBISAM (Btrieve) database to SQL Server, which had separate date and time columns where I wanted to import a DateTime.  Using the ODBC driver and a DataReader data source, it was returning the date as the DT\_DBTIMESTAMP type and the time as a DT\_I8 long integer.

I tried using the Derived Column transformation to combine them, but never could get it to work.  There simply aren&#8217;t many date functions supported by the Derived Column formulas.  Finally, I settled on using the following VBS script using the Script Component transformation:<!-- code formatted by http://manoli.net/csharpformat/ -->

```vb
If Not Row.StartDate_IsNull Then
    If Row.StartTime_IsNull Then
        Row.StartDateTime = Row.StartDate
    Else
        Row.StartDateTime = Row.StartDate.AddTicks(Row.StartTime)
    End If
End If
```