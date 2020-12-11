---
id: 312
title: C# 9 Records and Init Only Settings Without .NET 5
date: 2020-12-11T06:00:00-05:00
permalink: /csharp/2020/12/11/csharp-9-records-and-init-only-setters-without-dotnet5.html
categories:
  - C#
  - .NET Core
summary: Another year is starting soon, and we have another new version of C# with great new features to learn about. The question, as always, is what can we use while maintaining backward compatibility.
---
{: .notice--info}
This blog is one of The December 11th entries on the [2020 C# Advent Calendar](https://www.csadvent.christmas/). Thanks for having me again Matt!

Some of the biggest new features in [C# 9](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9) have got to be
[Records](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#record-types) and
[Init only setters](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#init-only-setters). But when can you safely
use these features? Officially, you can only use them if you are targeting [.NET 5](https://docs.microsoft.com/en-us/dotnet/core/dotnet-five).
However, many developers may not be ready to move to .NET 5 yet, especially since it isn't an LTS release. And what about NuGet
packages targeting .NET Standard or other versions of .NET?

## TL;DR

For internal projects and/or types, you can safely use records and init only setters to target any modern version of .NET. I've tested
.NET 4.8, .NET Core 2.1, and .NET Core 3.1. However, for any public members exposed via NuGet packages there are **lots** of caveats.

Note: Most of the other C# 9 features are fine to use, regardless of target frameworks or consumer C# versions:

- Top-level statements
- Switch expression pattern matching enhancements
- Native sized integers (syntactic sugar for IntPtr/UIntPtr)
- Target-typed new expression
- Static anonymous functions
- GetEnumerator extension methods
- Lambda discard patterns
- Attributes on local functions
- New features for partial methods

## What are Records?

There are plenty of [blog posts](https://anthonygiretti.com/2020/06/17/introducing-c-9-records/) out there that describe records and
what they do, so I won't dig in too deep here. In simple terms, they provide a lot of boilerplate code that allows a developer
to create immutable reference types with value equality, easy methods to clone the types with slightly different values, and more.

```csharp
public namespace MyProgram
{
  public record Person
  {
    public string? FirstName { get; init; }
    public string? LastName { get; init; }
  }

  public record PersonWithHeight : Person
  {
    public int HeightInInches { get; init; }

    public PersonWithHeight Grow(int inches) =>
      this with { HeightInInches = HeightInInches + inches };
  }
}

PersonWithHeight person = new()
{
  FirstName = "Brant",
  LastName = "Burnett",
  HeightInInches = 69
};
```

Under the covers, records are really classes with a lot of boilerplate code already added to deal with equality comparisons,
cloning, ToString, etc.

## What are init only setters?

Init only setters are not really specific to records, though they are quite powerful when combined. You can see an example
of init only setters above in the example for records. Instead of a `set` keyword, the property uses an `init` keyword.
These properties can only be set as part of an object initializer combined with the constructor, after which they may not be
modified.

```csharp

Person person = new()
{
  FirstName = "Brant" // Allowed
};

person.LastName = "Burnett"; // This will cause a compiler error
```

Under the covers, C# does this by making the setter method have a slightly different signature. Normally, the signature would
be `void set_FirstName(string value)`, but an init only setter has the signature
`void modreq(System.Runtime.CompilerServices.IsExternalInit) set_FirstName(string value)`. Note that `modreq` can't be
directly added to a method signature in C#, this is an IL construct. C# adds it for you in this case.

## Requirements to Create Records and Init Only Setters

### LangVersion 9

Your project must be using C# 9 or later. If you're targeting .NET 5, this should be the case already. However, if you
target other runtimes, you may need to manually enable C# 9 in your `.csproj` file.

```xml
<PropertyGroup>
  <LangVersion>9</LangVersion>
</PropertyGroup>
```

You must also be using Visual Studio 2019 16.8 or later, or MSBuild 2019 16.8 or later, or the .NET Core SDK 5.0.100 or
later. These are the versions that include the C# 9 compiler.

### Creating Init Only Properties on Older Frameworks

The key to init-only properties is the `IsExternalInit` class, which is basically nothing but a placeholder (somewhat like
an empty attribute) that is applied to the `void` return type of the setter. This type is defined as part of .NET 5, but
if you're not targeting .NET 5 then it's not available for the compiler to reference.

```text
CS0518 Predefined type 'System.Runtime.CompilerServices.IsExternalInit' is not defined or imported
```

To fix this, you must include the type yourself in your code. I recommend a copy/paste of the following into a file in your
project. It includes conditional compilation directives which work if you're multi-targeting so that it isn't included unless
necessary.

```csharp
// Licensed to the .NET Foundation under one or more agreements.
// The .NET Foundation licenses this file to you under the MIT license.
// See the LICENSE file in the project root for more information.

#if NETSTANDARD2_0 || NETSTANDARD2_1 || NETCOREAPP2_0 || NETCOREAPP2_1 || NETCOREAPP2_2 || NETCOREAPP3_0 || NETCOREAPP3_1 || NET45 || NET451 || NET452 || NET6 || NET461 || NET462 || NET47 || NET471 || NET472 || NET48

using System.ComponentModel;

// ReSharper disable once CheckNamespace
namespace System.Runtime.CompilerServices
{
    /// <summary>
    /// Reserved to be used by the compiler for tracking metadata.
    /// This class should not be used by developers in source code.
    /// </summary>
    [EditorBrowsable(EditorBrowsableState.Never)]
    internal static class IsExternalInit
    {
    }
}

#endif
```

### Creating Records on Older Frameworks

Records actually work fine on older frameworks without any special changes, so long as you aren't using init only setters.

However, records do transparently apply init only setters if you use this syntax to define the properties:

```csharp
public record Person(string FirstName, string LastName)
{
}
```

In that case, be sure to add the `IsExternalInit` class above.

## Consuming Records and Init Only Properties

What about including records and init only properties in things like NuGet packages? What are the requirements
for package consumers to be able to use these features?

### Private Members

It is safe to include `private` or `internal` records or properties with init only setters in public facing
assemblies. Records are basically just syntactic sugar on top of classes, and `modreq` used by init only setters
have been around a long time.

### Public Init Only Setters

To use a `public` or `protected` property with an init only setter, the consumer **must** be using a C# compiler that
supports C# 9. Even if they aren't using LangVersion 9 in their csproj, the compiler must be capable of understanding
the `modreq`. If the consumer doesn't understand the init only setter, it will be unable to access the setter at all
and will appear as a read-only property.

Similarly, other languages like VB.NET don't like init only setters. In fact, VB.NET won't recognize the property at
all, even as read-only, and even with the latest version of the VB.NET compiler.

As a result, I do not recommend using init only setters for public members unless it is for an internal project with
a known list of consumers. All of these consumers must be C# projects, and all developers must be using the latest
C# compiler.

### Public Records

The compatibility of records across different versions of C# is a bit more confusing. First of all, if you're using
init only setters on your records (you probably are) then see those rules above.

C# consumers can get most of the other features of records, except cloning. The clone method is hidden under the
special name `<Clone>$` and you can't reach it directly in C#, you must use the `with` operator which is only
available in C# 9.

The same limitation applies in VB.NET, except even on the latest version there is no equivalent of the `with`
operator. This makes cloning completely unavailable.

## Summary

Okay, all of that was pretty convoluted. Variables include your target framework, your consumer's target framework,
your LangVersion, and the version of MSBuild or .NET Core SDK installed on the consumer's development machine and
build agents.

For developers of NuGet packages which must be consumed by many users, here's a matrix that will hopefully make
things a bit clearer. These are my recommendations, some of these listed as Not Available could be considered
partially available. But in my opinion the limitations aren't worth it and it should be avoided.

| Private/Internal Members       | .NET 5.0 Target | Older Targets |
| ------------------------------ | --------------- | ------------- |
| C# .NET 5.0 Consumer           | Available       | Available (*) |
| VB .NET 5.0 Consumer           | Available       | Available (*) |
| C# 9 Older Framework Consumer  | n/a             | Available (*) |
| C# <9 Older Framework Consumer | n/a             | Available (*) |
| VB Older Framework Consumer    | n/a             | Available (*) |

| Public/Protected Members       | .NET 5.0 Target | Older Targets |
| ------------------------------ | --------------- | ------------- |
| C# .NET 5.0 Consumer           | Available       | Available (*) |
| VB .NET 5.0 Consumer           | Available       | Available (*) |
| C# 9 Older Framework Consumer  | n/a             | Available (*) |
| C# <9 Older Framework Consumer | n/a             | Not Available |
| VB Older Framework Consumer    | n/a             | Not Available |

(*) = Must add IsExternalInit class snippet above

## Conclusion

To boil it all down even more simply, here are my overall recommendations for when you should
and should not use records and init only setters.

| Project Type             | Public Records | Internal/Private Records | Public/Protected Init Only Setters | Internal/Private Init Only Setters |
| ------------------------ | -------------- | ------------------------ | ---------------------------------- | ---------------------------------- |
| Internal Use             | Yes            | Yes                      | Yes                                | Yes                                |
| NuGet                    | No             | Yes                      | No                                 | Yes                                |
| NuGet .NET 5 Target Only | Yes            | Yes                      | Yes                                | Yes                                |
