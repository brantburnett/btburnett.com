---
id: 311
title: IAsyncEnumerable<T> Is Your Friend, Even In .NET Core 2.x
date: 2019-11-27T06:00:00-05:00
permalink: /csharp/2019/12/01/iasyncenumerable-is-your-friend.html
categories:
  - C#
  - .NET Core
summary: IAsyncEnumerable is a great new feature in C# 8. This post should help clarify what they do, when to use them, and when they can be used in a particular project.
---
{: .notice--info}
This blog is one of The December 1st entries on the [2019 C# Advent Calendar](https://crosscuttingconcerns.com/The-Third-Annual-csharp-Advent). Thanks for having me again Matt!

My favorite new feature found in [C# 8](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8) has got to be
[Asynchronous Streams](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-8#asynchronous-streams), a.k.a.
Asynchronous Enumerables. However, I think there may some confusion as to what they do, when to use them, and even **if**
they can be used in a particular project. This post will hopefully clarify some of these points.

## TL;DR

You can use `IAsyncEnumable<T>` and the related C# 8 features in .Net Core 2.x or .NET Framework 4.6.1.

## Why use `IAsyncEnumerable<T>`?`

When writing efficient applications intended to handle large numbers of simultaneous operations, such as web applications,
blocking operations are the enemy. Any time an application is waiting on some kind of I/O-bound operation, such as a network
response or a hard disk read, it is always best to relinquish the thread back to the thread pool. This allows the CPU to
work on other operations while the method is waiting, and then continue the work once there is more to be done. Lots of details
about why are [available here](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-based-asynchronous-programming).

In many cases, it's efficient enough to simply return a regular `IEnumerable<T>` asynchronously, like so:

```cs
public async Task DoWork()
{
    var query = BuildMyQuery();

    var queryResult = await query.ExecuteAsync();

    foreach (var item in queryResult)
    {
        // Do work here
    }
}
```

However, this is may not be the most effective approach. There are two potential limitations, depending on the backing implementation of
`ExecuteAsync` in the example above.

1. The implementation of ExecuteAsync may return after it gets the first part of the data, or even none. Then, as it is enumerated it may block waiting for more items to arrive.
2. The implementation of ExecuteAsync may wait until it has **all** of the data before returning, delaying the point where DoWork may begin processing the data.

`IAsyncEnumerable<T>` and its sibling `IAsyncEnumerator<T>`, on the other hand, return a `Task` as the stream is iterated. This allows methods
such as `ExecuteAsync` to return early and then wait for more data as it is iterated, without blocking the thread.

## When **Not** To Use IAsyncEnumerable

Don't use `IAsyncEnumerablt<T>` for any collection which is inherently synchronous and CPU-bound.
For those cases, continuing using `IEnumerable<T>`. The extra overhead of handling asynchronous tasks
will actually be less performant in these scenarios.

Knowing which to use on a interface, which can have different backing implementations which may or may not be
synchronous, is a bit trickier. In this case, use the pattern for the most likely scenario. If in doubt, lean
towards `IAsyncEnumerable<T>` because it can be used for either scenario and is therefore more flexible.

## Returning an IAsyncEnumerable in C# 8

For simple use cases, returning an `IAsyncEnumerable<T>` is as easy as an `IEnumerable<T>`, using an iterator function.

```cs
public async IAsyncEnumerable<XYZ> GetXYZAsync() {
    for (var i=0; i<20; i++)
    {
        var item = await GetXYZById(i);

        yield return item;
    }
}
```

It is also possible to write an iterator method which returns `IAsyncEnumerator<T>`.

If the method should to support cancellation, the CancellationToken needs to be decorated with the
`EnumeratorCancellation` attribute:

```cs
public async IAsyncEnumerable<XYZ> GetXYZAsync([EnumeratorCancellation] CancellationToken cancellationToken = default) {
    for (var i=0; i<20; i++)
    {
        var item = await GetXYZById(i, cancellationToken);

        yield return item;
    }
}
```

## Consuming an IAsyncEnumerable in C# 8

To consume the IAsyncEnumerable, simply use the new `await foreach` statement within an asynchronous method.

```cs
await foreach (var item in GetXYZAsync())
{
    // do things here
}
```

To control the synchronization context, `ConfigureAwait` is available just like on `Task`.

```cs
await foreach (var item in GetXYZAsync().ConfigureAwait(false))
{
    // do things here
}
```

To support cancellation tokens:

```cs
await foreach (var item in GetXYZAsync().WithCancellation(cancellationToken))
{
    // do things here
}
```

## Special Compatibility Concerns

Depending on the type of project and the version of .NET being targeted, there may be concerns
about compatibility. One myth is that these features can only be used with .NET Core 3.0 and C# 8.

### Can I Use IAsyncEnumerable When Targeting .NET Core 2.x?

**Short answer**: Yes

To gain access to IAsyncEnumerable, install the compatibility NuGet package
[Microsoft.Bcl.AsyncInterfaces](https://www.nuget.org/packages/Microsoft.Bcl.AsyncInterfaces/). This provides
the types that are missing in .NET Standard 2.0.

However, producing and consuming IAsyncEnumerables is a bit more difficult. Here's an example consumer:

```cs
var enumerable = GetXYZAsync();
var enumerator = await enumerable.GetAsyncEnumerator();
try
{
    while (await enumerator.MoveNextAsync())
    {
        var item = enumerator.Current;

        // Do things here
    }
}
finally
{
    await enumerator.DisposeAsync();
}
```

To get around this limitation, there are ways to access C# 8 language features, like asynchronous streams, even from .NET Core 2.x.
The requirements are:

- Use .NET Core SDK 3.0 or MSBuild Tools 2019 as the compiler.
- Use Visual Studio 2019 or VSCode as the IDE.
- Add a `<LangVersion>8</LangVersion>` property to the project file.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <LangVersion>8</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Bcl.AsyncInterfaces" Version="1.0.0" />
  </ItemGroup>

</Project>
```

### Can I Use IAsyncEnumerable When Targeting .NET Framework

**Short answer**: Yes, 4.6.1 or later

.NET Framework 4.6.1 is compatible with .NET Standard 2.0. Therefore, so long as the project is targeting .NET Framework 4.6.1 or later,
the same rules apply as for .NET Core 2.x. Just follow the same steps as above.

### What About NuGet Packages?

**Short answer**: Yes, targeting .NET Standard 2.0 or later

When creating a NuGet package, maintaining backwards compatibility is key. This can make it difficult to use new features like
asynchronous streams. The good news is that there is a relatively easy path.

- Dual target .NET Standard 2.0 and 2.1.
- Consider targeting .NET 4.6.1 as well, see [Microsoft's guidance on cross platform targeting](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/cross-platform-targeting).
- Include [Microsoft.Bcl.AsyncInterfaces](https://www.nuget.org/packages/Microsoft.Bcl.AsyncInterfaces/) for the lower version targets only.
- Add a `<LangVersion>8</LangVersion>` property to the project file.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net461;netstandard2.0;netstandard2.1</TargetFrameworks>
    <LangVersion>8</LangVersion>
  </PropertyGroup>

  <ItemGroup Condition=" '$(TargetFramework)' == 'net461' Or '$(TargetFramework)' == 'netstandard2.0' ">
    <PackageReference Include="Microsoft.Bcl.AsyncInterfaces" Version="1.0.0" />
  </ItemGroup>

</Project>
```

However, this approach has problems if the package needs to target versions of .NET Core before 2.0, .NET Standard before 2.0,
or .NET Framework before 4.6.1. The Microsoft.Bcl.AsyncInterfaces package isn't compatible with frameworks prior these versions.
This can still be addressed by using preprocessor conditionals within the codebase to exclude `IAsyncEnumerable<T>` support,
but is much more cumbersome and outside the scope of this post.

## What about LINQ?

Most .NET developers love to use LINQ, either using the query syntax or using the functional
syntax like `.Where(p => p != null)` or `.Select(p => p.Property)`. Is it possible to use LINQ with `IAsyncEnumerable<T>`?

To get support for the functional LINQ syntax, such as `.Where(p => p != null)` or `.Select(p => p.Property)`, install
[System.Linq.Async](https://www.nuget.org/packages/System.Linq.Async/). This package is brought to us by the group that makes
[ReactiveX](http://reactivex.io/). Be sure to use at least version 4.0.0.

## Conclusion

Don't be afraid to use `IAsyncEnumerable<T>` in your code. It's really a lot more available than most developers think it is,
and is very easy to use.
