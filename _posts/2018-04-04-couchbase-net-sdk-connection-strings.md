---
id: 303
title: Couchbase .NET SDK Connection Strings
date: 2018-04-04T15:14:47+00:00
guid: http://btburnett.com/?p=303
permalink: /2018/04/couchbase-net-sdk-connection-strings.html
categories:
  - Couchbase
---
With the release of [Couchbase .NET SDK 2.5.9](https://www.nuget.org/packages/CouchbaseNetClient/2.5.9) there are new options available for configuring connections to [Couchbase](https://www.couchbase.com/) clusters.

## Server Lists

Having a robust Couchbase cluster means configuring a list of servers for bootstrapping from the client. This allows the system to bootstrap even if one of the nodes is offline. Previously, it was necessary to define these servers as a collection:

```xml
<couchbase username="user" password="password">
  <servers>
    <add uri="http://server1:8091/" />
    <add uri="http://server2:8091/" />
    <add uri="http://server3:8091/" />
  </servers>
</couchbase>
```

Or:

```json
{
  "Couchbase": {
    "username": "user",
    "password": "password",
    "servers": [
      "http://server1:8091/",
      "http://server2:8091/",
      "http://server3:8091/"
    ]
  }
}
```

## Connection Strings

While the "servers" collection is fully functional, it can make overriding settings across different deployment environments cumbersome. This is especially true for .NET Core, where individual settings will often be overridden using [environment variables](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?tabs=basicconfiguration#configuration-by-environment).

Version 2.5.9 of the Couchbase .NET SDK adds support for couchbase:// connection strings, which define all details about the servers in a single string. This makes per-environment overrides much easier:

```xml
<couchbase username="user" password="password" connectionString="couchbase://server1,server2,server3">
</couchbase>
```

Or:

```json
{
  "Couchbase": {
    "username": "user",
    "password": "password",
    "connectionString": "couchbase://server1,server2,server3"
  }
}
```

> **Note:** If you mix the old "servers" method with the "connectionString" method, the connection string will take precedence.
{: .notice--info}

## Connection String Syntax

The connection string supports 3 schemes:

* `couchbase://` &#8211; Bootstraps using CCCP with a fallback to HTTP
* `couchbases://` &#8211; The same as "couchbase://", except using TLS encryption
* `http://` &#8211; Legacy HTTP bootstrap

This is followed by one or more servers, separated by commas. Each server may have a port number as well, though port numbers are not required unless your cluster uses non-standard ports.

Examples:

* `couchbase://server1,10.0.0.10:11210,server3`
* `couchbases://server1:11207,server2:11207,server3:11207`
* `http://server1:8091`

## Using Connection Strings In Kubernetes

As a further example, here's how connection strings could be used when deploying a .NET Core service to Kubernetes:

First, create a secret with the credentials and connection string:

```sh
kubectl create secret generic couchbase \
  --from-literal=connectionString=couchbase://server1,server2,server3 \
  --from-literal=username=user \
  --from-literal=password=password
```

Then, when deploying the application, override the configuration using environment variables from the secret. The key values are in the "env:" tag of the YAML file below:

```yaml
kind: ReplicaSet
apiVersion: extensions/v1beta1
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: Production
        - name: Couchbase__connectionString
          valueFrom:
            secretKeyRef:
              name: couchbase
              key: connectionString
        - name: Couchbase__username
          valueFrom:
            secretKeyRef:
              name: couchbase
              key: username
        - name: Couchbase__password
          valueFrom:
            secretKeyRef:
              name: couchbase
              key: password
        resources:
          limits:
            cpu: 100m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: "/health"
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
```

Having only three environment variables to override (connectionString, username, and password) makes this approach much simpler than a list of servers.
