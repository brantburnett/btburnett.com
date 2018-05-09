---
id: 22
title: SQL Server 2005 Service Broker Error Code 25 (Event ID 28054)
date: 2008-09-02T12:06:00+00:00
guid: http://www.btburnett.com/?p=22
permalink: /2008/09/sql-server-2005-service-broker-error-code-25-event-id-28054.html
categories:
  - SQL Server
---
Another issue I've run into with the service broker and moving databases between servers is this error message:

```text
Error code:25. The master key has to exist and the service master key encryption is required.
```

The solution that I found was the regenerate the key for the database using the following code:

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '1ComplexPassword!'
```

You might also need to run the following afterwards:

```sql
ALTER DATABASE dbname SET NEW_BROKER.
```

Also, see my previous post [SQL Server 2005 Service Broker Error 15517](/2008/05/sql-server-2005-service-broker-error-15517.html) for another error message and solution.
