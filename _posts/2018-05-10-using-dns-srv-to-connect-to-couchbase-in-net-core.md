---
id: 303
title: Using DNS SRV to connect to Couchbase in .NET Core
date: 2018-05-10T09:45:00-04:00
categories:
  - Couchbase
---
With the release of [Couchbase.Extensions.DnsDiscovery 2.0.2](https://www.nuget.org/packages/Couchbase.Extensions.DnsDiscovery/), there are now new options availble for bootstrapping [Couchbase](https://www.couchbase.com/) using DNS SRV records in .NET Core.  The new approach works in tandem with version 2.5.9 of the Couchbase SDK and the new [connectionString based bootstrap method](/2018/04/couchbase-net-sdk-connection-strings.html).

The DnsDiscovery extension has been powering bootstrapping based on DNS SRV records for some time. The new release, however, makes the entire process much more streamlined.  First, setup an SRV record for your cluster.  For example, to run a cluster at `internal.mydomain.com`, add a SRV record for `_couchbase._tcp.internal.mydomain.com`.  The formatting of the prefix is important.  It must start with either `_couchbase._tcp` or `_couchbases._tcp` (for SSL).  The record itself would look like this:

```text
10 10 11210 node1.internal.mydomain.com.
10 10 11210 node2.internal.mydomain.com.
10 10 11210 node3.internal.mydomain.com.
```

The first two columns are not used.  The third column is the port number.  11210 is the standard Couchbase port, but use 11207 for the  `couchbases` SSL encyrpted protocol.  There is one line in the record for each node in the Couchbase cluster.

Next, simply register the DnsDiscovery extension in ConfigureServices:

```cs
public void ConfigureServices(IServiceCollection services)
{
    // Register other services here

    services
        .AddCouchbase(options =>
        {
            Configuration.GetSection("Couchbase").Bind(options);
        })
        .AddCouchbaseDnsDiscovery(); // Add this line, no parameters
}
```

Finally, in `appsettings.json`, or any other configuration source, set the Couchbase connectionString to reference the domain.  The string should include only a single domain, and no port number.

```json
{
    "Couchbase": {
        "connectionString": "couchbase://internal.mydomain.com",
        "username": "myuser",
        "password": "password"
    }
}
```

The DnsDiscovery extension will detect that the connection string may be an SRV record, and resolve it automatically.  However, it is perfectly safe to substitute other types of connection strings in other environments.  If the string has multiple servers or port numbers, DnsDiscovery ignores the connection string.  The same is true if there is no SRV record found.

This approach to bootstrapping brings complete flexibility for configuring access to your Couchbase cluster in every environment using a single, easy-to-configure string.
