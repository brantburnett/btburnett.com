---
id: 182
title: Testing an SDK for Async/Await SynchronizationContext Deadlocks
date: 2016-04-27T14:59:18+00:00
guid: http://btburnett.com/?p=182
permalink: /2016/04/testing-an-sdk-for-asyncawait-synchronizationcontext-deadlocks.html
categories:
  - .NET Framework
---
The purpose of this post is to explain how to write unit and/or integration tests that ensure you don't have synchronization context deadlocks in an SDK. A very detailed explanation of the problem and the solution can be found [here](http://blog.stephencleary.com/2012/07/dont-block-on-async-code.html). However, before explaining how to write the tests, I will give a brief summary of the problem and the solutions. Then we'll get into how to test to make sure that the solution is correctly implemented now and won't regress in the future.

## The Problem

One of the common API developer pitfalls in asynchronous development with the .Net TPL is deadlocks. Most commonly, this is caused by SDK consumers using your asynchronous SDK in a synchronous manner. For example:

```cs
public ActionResult Index()
{
    // Note to consumers: Where possible DON'T DO THIS.  Just make your MVC action async, it works MUCH better.
    var data = api.SomeActionAsync().Result;

    return View(data);
}
```

The example above is typically a problem because MVC runs actions inside a [SynchronizationContext](https://msdn.microsoft.com/en-us/library/system.threading.synchronizationcontext(v=vs.110).aspx). The MVC synchronization context prevents more than one thread from operating within the context simultaneously. The process flow works as follows:

1. Thread A runs the action above, and requests "SomeActionAsync"
2. Thread A blocks waiting on "Result" for "SomeActionAsync"
3. Thread B begins processing the work for "SomeActionAsync"
4. At some point, Thread B attempts to synchronize onto the SynchronizationContext from the MVC action, and is blocked waiting for Thread A to release it.
5. We have a deadlock!

So why does Step 4 above happen? I know I didn't write any code that requested that the MVC SynchronizationContext be used! Well, if you are using the async/await programming model, you're doing so without even knowing it.

<pre class="brush: csharp; title: ; notranslate" title="">public async Task&lt;SomeResult&gt; SomeActionAsync()
{
    // do some work

    var temp = await obj.SomeOtherActionAsync();

    // do more work

    return result;
}
</pre>

The await above automatically tries to synchronize with the SynchronizationContext. This is actually pretty important for writing async MVC actions. When an async method is awaited in the action, once it's complete we really want the remainder of the action to run within the SynchronizationContext. But within our SDK, we probably don't want that to happen because it usually doesn't have any value to us.

## The Solution

There are two solutions to this problem.  The good one, and the bad one.

## The Bad Solution: Require that SDK consumers use the SDK asynchronously

I don't like this solution because it's difficult to ensure that consumers use it correctly. It's also a barrier to SDK use. I really believe that consumers should be given the option to consume the SDK how they like, even if it's a bad way to consume it. The TPL provides a .Result call, so where possible we should make it work.

```cs
public async Task<ActionResult> Index()
{
    var data = await api.SomeActionAsync();

    return View(data);
}
```

An important note for SDK consumers, though. For you, this is the Good Solution. You should always use asynchronous API calls in an asynchronous manner whenever possible. This is only a Bad Solution if the SDK developer is assuming that you always do this.

## The Good Solution: Fix the problem on the SDK side

Thankfully, the TPL provides us with a simple workaround in the SDK, using ConfigureAwait(false). Calling this routine before awaiting a task will cause it to ignore the SynchronizationContext.

```cs
public async Task<SomeResult> SomeActionAsync()
{
    // do some work

    var temp = await obj.SomeOtherActionAsync().ConfigureAwait(false);

    // do more work

    return result;
}
```

## Most Importantly, The Test

The problem with the Good Solution is that it requires you to place a **lot** of ConfigureAwait(false) calls throughout the SDK. This can be cumbersome and easy to forget. Though there is a Resharper plugin to help, [ConfigureAwaitHelper](https://resharper-plugins.jetbrains.com/packages/ConfigureAwaitChecker.v9/).

Any good SDK comes with a battery of unit and integration tests. So the trick is to add tests to the SDK that will ensure that we don't forget any ConfigureAwait(false) calls. So how do we add a test or tests that ensure we called ConfigureAwait(false)?

The trick is understanding how the SynchronizationContext works. Anytime it is used, a call will be made to either it's Send or Post method. So, all we need to do is make a mock and ensure that these methods never get called:

```cs
[Test]
public void Test_SomeActionNoDeadlock()
{
    // Arrange
    var context = new Mock&lt;SynchronizationContext&gt;
    {
        CallBase = true
    };

    // Do other arrange actions here

    SynchronizationContext.SetSynchronizationContext(context.Object);
    try
    {
        // Act
        class.SomeActionAsync().Wait();

        // Assert

        // If the method is incorrectly awaiting on the current SynchronizationContext
        // We will see calls to Post or Send on the mock

        context.Verify(m =&gt; m.Post(It.IsAny&lt;SendOrPostCallback&gt;(), It.IsAny&lt;object&gt;()), Times.Never);
        context.Verify(m =&gt; m.Send(It.IsAny&lt;SendOrPostCallback&gt;(), It.IsAny&lt;object&gt;()), Times.Never);
    }
    finally
    {
        SynchronizationContext.SetSynchronizationContext(null);
    }
}
```

This example uses NUnit and Moq, but it should work just as well with other testing frameworks. Now we have a way to guarantee that ConfigureAwait(false) was used throughout the SDK method, so long as we get 100% test coverage through the logical paths in the method.

Of course, you may ask "Why do I need this?  I just looked at the code, I'm always calling ConfigureAwait(false)!" The answer is preventing regressions. You might have remembered today, but next month when you're making a change it's **very** easy to forget. This test is your fallback plan in case you make a mistake in the future.
