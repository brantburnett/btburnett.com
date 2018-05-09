---
id: 23
title: Extracting The Date From a SQL DateTime In SQL Queries
date: 2008-09-30T06:06:00+00:00
guid: http://www.btburnett.com/?p=23
permalink: /2008/09/extracting-the-date-from-a-sql-datetime-in-sql-queries.html
categories:
  - SQL Server
---
Ever wondered how to quickly and easily get the date portion of a datetime field in a SQL query? Seems like it should be easy, right? Unfortunately, it isn't. I've found a lot of solutions, but this one is the most elegant I've seen.

```sql
CONVERT(datetime, FLOOR(CONVERT(float, DateTime)))
```

I've put this into a user-defined function below:

```sql
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[ExtractDate]
    (@DateTime datetime)
RETURNS datetime
AS
BEGIN
    RETURN CONVERT(datetime, FLOOR(CONVERT(float, @DateTime)))
END
```

This can be called like so:

```sql
SELECT dbo.ExtractDate(DateTime) FROM Table
```

Credit for this conversion method goes to [Matt Berseth](http://mattberseth.com/), i stumbled across it in the example code for something else.
