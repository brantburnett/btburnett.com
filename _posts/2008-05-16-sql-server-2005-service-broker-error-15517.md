---
id: 12
title: SQL Server 2005 Service Broker Error 15517
date: 2008-05-16T09:27:00+00:00
guid: http://www.btburnett.com/?p=12
permalink: /2008/05/sql-server-2005-service-broker-error-15517.html
categories:
  - SQL Server
---
Sometimes I've run into the following error in the event log when using the service broker to do SQL command notifications with SQL 2005:

```text
An exception occurred while enqueueing a message in the target queue. Error: 15517, State: 1. Cannot execute as the database principal because the principal "dbo" does not exist, this type of principal cannot be impersonated, or you do not have permission.
```

It tends to fill up the event log pretty badly when you see it. I've especially seen it when I move the database from one server to another. To fix it, use this command:

```sql
ALTER AUTHORIZATION ON DATABASE::[dbname] TO [SA]
```

This should fix it for you.
