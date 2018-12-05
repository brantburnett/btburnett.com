---
id: 309
title: A Restful Job Pattern For A C# Microservice
date: 2018-12-06T07:30:00-05:00
categories:
  - Couchbase
  - Microservices
  - REST
---
{: .notice--info}
This blog is one of The December 6th entries on the [2018 C# Advent Calendar](https://crosscuttingconcerns.com/The-Second-Annual-C-Advent). Thanks for having me again Matt!

One of the key differences between REST (Representational State Transfer) and RPC (Remote Procedure Calls) is that all actions within REST are conducted as if getting or mutating a specific resource. In most cases, especially basic CRUDL operations, this makes for a cleaner and more concise API structure than RPC. However, it can also add friction in certain cases.

One such case is dealing with long-running jobs. What if creating a new resource takes time and isn't immediately available? I won't go into the details in this post, [here is a post by Victor Farazdagi that provides an overview](https://farazdagi.com/2014/rest-and-long-running-jobs/). However, I'm going to use the basic endpoints he discusses in my examples.

This concept is great, but what about the practical implementation? If your application is a based on a single, long-lived server that never fails or reboots, read no further because the implementation is easy. If, on the other hand, your application operates in the real world of ephemeral servers, load balancers, microservices, and unreliable hardware, keep reading!

{: .notice--info}
**Note:** The completed and runnable example is available at [https://github.com/brantburnett/CouchbaseRestfulJobPattern](https://github.com/brantburnett/CouchbaseRestfulJobPattern).

## The Architecture

Before I dig into an implementation, let's discuss the high-level architecture of the application.

- The application is a microservice running in a Docker container
- There will be multiple instances of the application behind a load balancer
- Any given instance may die at any given time due to hardware failure, scale-in, etc.
- The instances all share a data store (in this case [Couchbase](https://www.couchbase.com/), but any data store supporting atomic writes would suffice)
- The instances are connected by a message bus which supports At Least Once delivery (such as [RabbitMQ](https://www.rabbitmq.com/), [Kafka](https://kafka.apache.org/), etc)

I'm also going to assume that we already have the basic application in place with endpoints for managing stars, but it currently doesn't use the job pattern.

## The Requirements

1. When a star is created, we should return quickly with a link to a long-running job
2. There should be a way to monitor the job for completion and retrieve to star ID
3. Jobs should trigger quickly under normal circumstances
4. Job processing load should be at least somewhat balanced across application instances
5. Jobs should only run on one application instance at a time
6. An application instance shouldn't oversubscribe to too many jobs at once
7. Job processing should not prevent the application from exiting during scale-in or maintenance
8. Jobs should be resumed on another application instance if an instance is stopped or fails

## The Job Document

First, let's create a `JobRepository` that can be used to persist the job and its status to the database. We'll support document expirations on jobs so that old, completed jobs can be cleaned up eventually.

```cs
[DocumentTypeFilter(TypeString)]
public class Job
{
    private const string TypeString = "job";

    public long Id { get; set; }
    public string Type => TypeString;

    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public Star CreateStar { get; set; }

    [JsonConverter(typeof(StringEnumConverter))]
    public JobStatus Status { get; set; }

    public static string GetKey(long id) => $"{TypeString}-{id}";
}

public class JobRepository
{
    private readonly IBucket _bucket;

    public JobRepository(IDefaultBucketProvider bucketProvider)
    {
        _bucket = bucketProvider.GetBucket();
    }

    public Task<IEnumerable<Job>> GetAllJobsAsync()
    {
        var context = new BucketContext(_bucket);

        return context.Query<Job>()
            .OrderBy(p => p.Id)
            .ExecuteAsync();
    }

    public Task<IEnumerable<Job>> GetIncompleteJobsAsync()
    {
        var context = new BucketContext(_bucket);

        return context.Query<Job>()
            .Where(p => p.Status != JobStatus.Complete)
            .OrderBy(p => p.Id)
            .ExecuteAsync();
    }

    public async Task<Job> GetJobAsync(long id)
    {
        var result = await _bucket.GetDocumentAsync<Job>(Job.GetKey(id));
        if (result.Status == ResponseStatus.KeyNotFound)
        {
            return null;
        }

        // Throw an exception on a low-level error
        result.EnsureSuccess();

        return result.Content;
    }

    public async Task CreateJobAsync(Job job)
    {
        if (job == null)
        {
            throw new ArgumentNullException(nameof(job));
        }

        job.Id = await GetNextJobIdAsync();

        var document = new Document<Job>
        {
            Id = Job.GetKey(job.Id),
            Content = job
        };

        var result = await _bucket.InsertAsync(document);
        result.EnsureSuccess();
    }

    public async Task UpdateJobAsync(Job job, TimeSpan? expiration)
    {
        if (job == null)
        {
            throw new ArgumentNullException(nameof(job));
        }

        var document = new Document<Job>
        {
            Id = Job.GetKey(job.Id),
            Expiry = (uint) (expiration?.TotalMilliseconds ?? 0),
            Content = job
        };

        var result = await _bucket.ReplaceAsync(document);
        result.EnsureSuccess();
    }

    private async Task<long> GetNextJobIdAsync()
    {
        // Redacted here for clarity
    }
}
```

Next, let's create `JobService` to handle business logic associated with these jobs (we'll be fleshing this out more later).

```cs
public class JobService
{
    private readonly JobRepository _jobRepository;


    public JobService(JobRepository jobRepository)
    {
        _jobRepository = jobRepository ?? throw new ArgumentNullException(nameof(jobRepository));
    }

    public async Task<Job> CreateStarJobAsync(Star star)
    {
        var job = new Job
        {
            CreateStar = star,
            Status = JobStatus.Queued
        };

        await _jobRepository.CreateJobAsync(job);

        return job;
    }
}
```

Next, we need a simple controller to get Job status.

```cs
[Route("job")]
[ApiController]
public class JobsController : ControllerBase
{
    private readonly JobRepository _jobRepository;
    private readonly IMapper _mapper;

    public JobsController(JobRepository jobRepository, IMapper mapper)
    {
        _jobRepository = jobRepository ?? throw new ArgumentNullException(nameof(jobRepository));
        _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
    }

    // GET job
    [HttpGet]
    public async Task<ActionResult<IEnumerable<JobDto>>> Get()
    {
        var jobs = await _jobRepository.GetAllJobsAsync();

        return jobs.Select(p => _mapper.Map<JobDto>(p)).ToList();
    }

    // GET job/5
    [HttpGet("{id}")]
    public async Task<ActionResult<JobDto>> Get(long id)
    {
        var result = await _jobRepository.GetJobAsync(id);
        if (result == null)
        {
            return NotFound();
        }

        return _mapper.Map<JobDto>(result);
    }
}
```

And, finally, we need to modify `StarsController` to create the job when a request is made, and return 202 Accepted as the status with a Location header.

```cs
[HttpPost]
public async Task<IActionResult> Post([FromBody] StarDto star)
{
    if (!ModelState.IsValid)
    {
        return BadRequest();
    }

    star.Id = 0;
    var job = await _jobService.CreateStarJobAsync(_mapper.Map<Star>(star));

    return AcceptedAtAction("Get", "Jobs", new {id = job.Id});
}
```

## Performing The Action

As an observant reader, I'm sure you noticed that we actually aren't doing anything with the job yet. It's just storing the Job document in Couchbase, but it never actually processes the job and the `status` will remain `Pending`.

For the next step, we need to make use of our message bus. Of course, we could just start the job on a background thread as part of `CreateStarJobAsync`. However, this could result in unbalanced load. Due to requirement #6, we're going to eventually put limiters in place so that an application instance won't process too many jobs at once. What would happen if the instance that received the POST request was already at the limit, but some other instance still had available capacity? We'd prefer for the work to be picked up on the other instance. Starting the job via the message bus separates the receiving instance from the processing instance, but still maintains a fast response time for requirement #3.

At this point, I'll assume that the application has a simple `MessageBus` class that acts as an abstraction for sending and receiving these messages. First, we add some logic to `JobService`.

```cs
public async Task<Job> CreateStarJobAsync(Star star)
{
    var job = new Job
    {
        CreateStar = star,
        Status = JobStatus.Queued
    };

    await _jobRepository.CreateJobAsync(job);

    // We're adding this line to the existing method
    QueueJob(job.Id);

    return job;
}

// And adding these methods below

public void QueueJob(long id)
{
    _messageBus.SendMessage(new Message {JobId = id});
}

public async Task ProcessNextJobAsync(CancellationToken cancellationToken)
{
    var message = await _messageBus.ReceiveMessage(cancellationToken);
    if (message != null)
    {
        await ExecuteJobAsync(message.JobId, cancellationToken);
    }
}

public async Task ExecuteJobAsync(long id, CancellationToken token)
{
    // Reload the document to make sure it's still pending
    var job = await _jobRepository.GetJobAsync(id);
    if (job.Status == JobStatus.Complete)
    {
        return;
    }

    // Update the status to Running
    job.Status = JobStatus.Running;
    await _jobRepository.UpdateJobAsync(job, null);

    // We're just emulating a long running job here, so just delay
    await Task.Delay(TimeSpan.FromSeconds(45), token);

    // To emulate a failed job, either throw an exception here
    // Or stop the app before the delay above is reached

    // Finish creating the star
    await _starRepository.CreateStarAsync(job.CreateStar);

    // Update the job status document
    job.Status = JobStatus.Complete;
    await _jobRepository.UpdateJobAsync(job, TimeSpan.FromDays(1));
}
```

Now, we add a `JobProcessor` class that loops and calls `ProcessNextJobAsync` until disposed.

```cs
public class JobProcessor : IDisposable
{
    private readonly JobService _jobService;
    private readonly ILogger<JobProcessor> _logger;

    private readonly CancellationTokenSource _cts = new CancellationTokenSource();

    private bool _started;
    private bool _disposed;

    public JobProcessor(JobService jobService, ILogger<JobProcessor> logger)
    {
        _jobService = jobService ?? throw new ArgumentNullException(nameof(jobService));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public void Start()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(nameof(JobProcessor));
        }

        if (!_started)
        {
            Task.Run(Poll);

            _started = true;
        }
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            _cts.Cancel();

            _disposed = true;
        }
    }

    private async Task Poll()
    {
        while (!_cts.IsCancellationRequested)
        {
            try
            {
                await _jobService.ProcessNextJobAsync(_cts.Token);
            }
            catch (OperationCanceledException)
            {
                // Ignore
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled exception in JobPoller");
            }
        }
    }
}
```

And, finally, we start the poller when the application starts and dispose it when stopping by adding these lines to `Startup` in the `Configure` method.

```cs
var jobProcessor = app.ApplicationServices.GetRequiredService<JobProcessor>();
jobProcessor.Start();

var appLifetime = app.ApplicationServices.GetRequiredService<IApplicationLifetime>();
appLifetime.ApplicationStopping.Register(() =>
{
    jobProcessor.Dispose();
});
```

Now we've addressed requirement #3 (start processing quickly), requirement #4 (load balance the processing across instances), and requirement #7 (the ApplicationStopping event will dispose the JobProcessor which then cancels any current processing via the CancellationToken).

## Preventing Duplicate Processing

When working with an At Least Once message bus, there's the possibility that two instances may receive the same message. Additionally, when we implement resumption of failed jobs for requirement #8 there will be an even greater risk of duplicate processing. However, having the same job processed at the same time by more than one instance is usually a bad thing.

We need to use a distributed [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) to ensure that only one instance may be working on a job at a time. When we start processing the job, we'll request a mutex. If it's denied, then the job is in process somewhere else and we skip it. If the mutex is issued, we hold it until we get done processing the job.

**Note:** This logic uses the [Couchbase.Extensions.Locks](https://www.nuget.org/packages/Couchbase.Extensions.Locks) package, which [I've blogged about before]({% link _posts/2018-11-04-couchbaselocks.md %}).

First, we add this method to `JobRepository`:

```cs
public Task<ICouchbaseMutex> LockJobAsync(long id, TimeSpan expiration)
{
    return _bucket.RequestMutexAsync(Job.GetKey(id), expiration);
}
```

Then, we alter the `ExecuteJobAsync` method in `JobService` to wrap the work in the mutex.

```cs
public async Task ExecuteJobAsync(long id, CancellationToken token)
{
    try
    {
        using (var mutex = await _jobRepository.LockJobAsync(id, TimeSpan.FromMinutes(1)))
        {
            mutex.AutoRenew(TimeSpan.FromSeconds(15), TimeSpan.FromHours(1));

            // Once we have the lock, reload the document to make sure it's still pending
            var job = await _jobRepository.GetJobAsync(id);
            if (job.Status == JobStatus.Complete)
            {
                return;
            }

            // Update the status to Running
            job.Status = JobStatus.Running;
            await _jobRepository.UpdateJobAsync(job, null);

            // We're just emulating a long running job here, so just delay
            await Task.Delay(TimeSpan.FromSeconds(45), token);

            // To emulate a failed job, either throw an exception here
            // Or stop the app before the delay above is reached

            // Finish creating the star
            await _starRepository.CreateStarAsync(job.CreateStar);

            // Update the job status document
            job.Status = JobStatus.Complete;
            await _jobRepository.UpdateJobAsync(job, TimeSpan.FromDays(1));
        }
    }
    catch (CouchbaseLockUnavailableException)
    {
        // Ignore and skip processing
    }
}
```

**Note:** It's particularly important to reload the Job from the data store and check its status before processing. This should be done within the mutex in case another instance is finishing the job just as this instance is trying to start the job.

## Preventing Oversubscription

Requirement #6 states that a single application instance shouldn't oversubscribe and process too many jobs at once. If an instance were to do this, too many CPU cycles could be spent on job processing to the detriment of HTTP response time.

The current implementation already limits an instance to one job at a time, but let's say we wanted to make it 2. This change to `JobProcessor` solves it.

```cs
private readonly SemaphoreSlim _concurrencyLimiter = new SemaphoreSlim(2);

private async Task Poll()
{
    while (!_cts.IsCancellationRequested)
    {
        try
        {
            await _concurrencyLimiter.WaitAsync(_cts.Token);

#pragma warning disable 4014
            _jobService.ProcessNextJobAsync(_cts.Token)
                .ContinueWith(t =>
                {
                    _concurrencyLimiter.Release();

                    if (t.IsFaulted)
                    {
                        _logger.LogError(t.Exception, "Unhandled exception in JobPoller");
                    }
                }, _cts.Token);
#pragma warning restore 4014
        }
        catch (OperationCanceledException)
        {
            // Ignore
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception in JobPoller");
        }
    }
}
```

Of course, better yet the limit should be configurable using `IOptions<T>`!

## Resuming Jobs

At this point, the only requirement we're missing is #8, resuming failed jobs on another instance. However, it's one of the most important requirements! Modern cloud infrastructures must be built to be highly fault tolerant, and should built on the premise of *when* a failure happens, not *if*.

To address resuming, we'll have a single application instance run a query every minute looking for jobs that are not flagged as `Completed`. It will test for a mutex on each job to see if it's already running, and if not refire the message into the message bus. A mutex issued to a failed application instance will eventually expire, causing the message to fire and another available instance to resume the work.

We'll also use another mutex for this process itself, so that only one instance tries it every minute. This will prevent every instance from firing multiple events for the same job at the same time.

```cs
public class JobRecoveryPoller : IDisposable
{
    private static readonly TimeSpan PollInterval = TimeSpan.FromMinutes(1);

    private readonly JobService _jobService;
    private readonly JobRepository _jobRepository;
    private readonly ILogger<JobRecoveryPoller> _logger;
    private readonly IBucket _bucket;

    private readonly CancellationTokenSource _cts = new CancellationTokenSource();

    private bool _started;
    private bool _disposed;


    public JobRecoveryPoller(JobService jobService, JobRepository jobRepository, IDefaultBucketProvider bucketProvider,
        ILogger<JobRecoveryPoller> logger)
    {
        _jobService = jobService ?? throw new ArgumentNullException(nameof(jobService));
        _jobRepository = jobRepository ?? throw new ArgumentNullException(nameof(jobRepository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));

        _bucket = bucketProvider.GetBucket();
    }

    public void Start()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(nameof(JobProcessor));
        }

        if (!_started)
        {
            Task.Run(Poll);

            _started = true;
        }
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            _cts.Cancel();

            _disposed = true;
        }
    }

    public async Task Poll()
    {
        while (!_cts.IsCancellationRequested)
        {
            try
            {
                // Wait for the poll interval
                await Task.Delay(PollInterval, _cts.Token);

                // Take out a lock for job recovery polling
                // Don't dispose so that it holds until it expires after PollInterval
                // This will ensure only one instance of the app polls every poll interval
                await _bucket.RequestMutexAsync("jobRecoveryPoller", PollInterval);

                var jobs = await _jobRepository.GetIncompleteJobsAsync();
                foreach (var job in jobs)
                {
                    try
                    {
                        // Try to lock the job to see if it's being processed currently
                        using (_jobRepository.LockJobAsync(job.Id, TimeSpan.FromSeconds(1)))
                        {
                        }

                        // Make sure we've unlocked the job before we get here
                        // And fire events into the message bus for the unhandled job
                        // This allows any instance with capacity to pick up the job
                        _jobService.QueueJob(job.Id);
                    }
                    catch (CouchbaseLockUnavailableException)
                    {
                        // Job is already being processed, ignore
                    }
                }
            }
            catch (OperationCanceledException)
            {
                // Ignore
            }
            catch (CouchbaseLockUnavailableException)
            {
                // Unable to lock for singleton job recovery poller process, ignore
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled exception in JobRecoveryPoller");
            }
        }
    }
}
```

We just need to alter our startup code again to start and dispose the recovery poller.

```cs
var jobProcessor = app.ApplicationServices.GetRequiredService<JobProcessor>();
jobProcessor.Start();

var jobRecoveryPoller = app.ApplicationServices.GetRequiredService<JobRecoveryPoller>();
jobRecoveryPoller.Start();

var appLifetime = app.ApplicationServices.GetRequiredService<IApplicationLifetime>();
appLifetime.ApplicationStopping.Register(() =>
{
    jobProcessor.Dispose();
    jobRecoveryPoller.Dispose();
});
```

## Conclusion

As you can see, handling long running jobs in a microservice architecture can be a bit tricky. Hopefully, having this pattern to use as a starting point will make things easier for you.

Here are a few improvements to consider in a real world implementation. I left these off intentionally so that I could focus on the key aspects of the requirements.

- Support for multiple different types of jobs
- Interfaces for the repositories and services at allow for mocking in unit tests
- An actual message bus (duh!)
- Put the pattern into a reusable Nuget package (If there's enough interest, I might do this myself)

If anyone sees any problems with my code or approach, please let me know so I can make updates.

{: .notice--info}
**Note:** The completed and runnable example is available at [https://github.com/brantburnett/CouchbaseRestfulJobPattern](https://github.com/brantburnett/CouchbaseRestfulJobPattern).
