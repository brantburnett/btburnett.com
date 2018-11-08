---
id: 307
title: Concurrency, Locks, the Cloud, and Couchbase
date: 2018-11-04T11:30:00-04:00
categories:
  - Couchbase
---
In an application that supports parallel processing, concurrency management is almost invariably required. There are many forms of parallel processing, including multithreading, multiple processes, or multiple servers. In the case of multithreading, different languages/frameworks will generally provide highly efficient in-process locks. For multiple process scenarios, such as NodeJS using the cluster module, there are various options which use interprocess communication.

## What About Multiple Servers?

These days it feels like every application is designed to run in the cloud. Applications designed for the cloud are built, from the ground up, to support running multiple copies of the application on multiple servers. Or at least, they should be. If you're building a cloud app that isn't designed for horizontal scaling, I'll go out on a limb and say you're doing it wrong.

So how should a distributed application handle locks and concurrency? Many scenarios can still take advantage of in-process locks. For example, [Linq2Couchbase](https://github.com/couchbaselabs/Linq2Couchbase) uses in-process locks to cache reflection results to improve performance. Since the cache is kept separately in-process on each server, the locks may be in-process. In fact, they should be in-process whenever possible to maintain performance.

However, these are not the only concurrency scenarios. In my work on microservices and other cloud architectures, I have frequently encountered scenarios where distributed locks are preferable or required. For example:

* Refreshing/managing shared caches stored in a cache cluster. If there are multiple requests that miss the cache at the same time, it may be desirable to have only one server/request refresh the cache to keep load low.
* Managing distributed jobs. For example, only one server in the cluster may run a job at once, but the servers "negotiate" which server runs the job via locks. Lock expiration may also be used to resume the job if a server fails unexpectedly.
* Scheduled task management. A task may need to run once an hour, but only one server in your cluster should execute the task.

## Distributed Locks and Couchbase

There are many solutions available for distributed locks, but at [CenterEdge Software](https://centeredgesoftware.com/) we specifically wanted an option powered by Couchbase. We already use Couchbase as our primary data store in the cloud, so using Couchbase avoids the need to spin up an additional HA cluster just to manage locks.

Couchbase's [CP architecture](https://en.wikipedia.org/wiki/Couchbase_Server#Architecture) makes it ideal for managing distributed locks. Because each document has a single server responsible for it, and updates are atomic, we can safely assume that two servers cannot mutate the same document simultaneously. There may be edge cases during server failover, but during normal operations, or even rebalance operations, consistency will be maintained.

## Couchbase.Extensions.Locks

To help manage distributed locks in Couchbase, I created the [Couchbase.Extensions.Locks](https://www.nuget.org/packages/Couchbase.Extensions.Locks/1.0.0-beta) NuGet package. This package has kindly been adopted by the Couchbase SDK team (thanks guys!) so that all C# developers can take advantage of this lightweight abstraction layer. It's currently in beta to give the community a chance to perform testing, but it is running in production successfully at CenterEdge Software today.

At this time, Couchbase.Extensions.Locks only supports mutexes, a.k.a. [Mutual Exclusion](https://en.wikipedia.org/wiki/Mutual_exclusion). This assures developers that only one thread of one application instance is within a critical section at a time. Future iterations may add other forms of concurrency control.

The supplied mutexes also require an expiration. This protects against indefinite locks due to application failure. There are built-in utilities to assist with renewing the mutex if your process is long running.

```cs
using (var mutex = await bucket.RequestMutexAsync("my-lock-name", TimeSpan.FromSeconds(10) /* expiration */))
{
    mutex.AutoRenew(TimeSpan.FromSeconds(5) /* renewal interval */, TimeSpan.FromMinutes(1) /* maximum lifetime */);

    while (true)
    {
        // Do some work here

        // This lock will be held until the end of the using statement,
        // so long as the total loop time is less than one minute
        // If the application crashes, the lock will be released within 10 seconds.
    }
}
```

Complete [instructions are available in the GitHub repository](https://github.com/couchbaselabs/Couchbase.Extensions/blob/master/docs/locks.md), including several usage patterns. This includes a [pattern for retries](https://github.com/couchbaselabs/Couchbase.Extensions/blob/master/docs/locks.md#retrying-locks) in case the lock is unavailable initially. In this case I chose a pattern using [Polly](https://github.com/App-vNext/Polly), I intentionally did not build the pattern into the Locks package. This allows developers to choose their favorite retry pattern and library, without introducing a specific dependency into the Locks package.

Using an memcached bucket for locks, or ephemeral for Couchbase Server Enterprise Edition 5.0 or later, is preferable. There generally isn't a need for lock documents to be persisted to disk, so this avoids unnecessary overhead on the cluster. If the bucket is configured for NRU (not recently used) ejection, make sure the locks are short-lived or have a fast renewal rate. This will ensure they aren't ejected, thereby releasing the lock, unexpectedly.

## Conclusion

For C# developers, there is now a standard package which delivers simple and lightweight distributed mutexes via a Couchbase cluster. I'm hopeful that that developers using other languages, such as Node or Go, may make similar packages using the same document schema. This would allow locks to be shared between any microservice in an application, regardless of language.

Please feel free to [file GitHub issues](https://github.com/couchbaselabs/Couchbase.Extensions/issues) if you find any bugs or have suggestions, and let me know on [Twitter](https://twitter.com/btburnett3) if you like it! We'll probably roll an official release after we've collected some feedback and fixed any issues.
