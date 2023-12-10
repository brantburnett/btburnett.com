---
id: 316
title: Generic Type Construction With Static Virtual Interface Members
date: 2022-12-12T00:00:00-05:00
permalink: /csharp/2023/12/12/generic-type-construction-with-static-virtual-interface-members
categories:
  - C#
  - .NET
summary: "I'd like to demonstrate a different use case for static virtual interface members: generic type construction."
---
{: .notice--info}
This blog is one of The December 12th entries on the [2023 C# Advent Calendar](https://www.csadvent.christmas/). Thanks for having me again!

Typically for C# Advent I write about a new C# feature. And there are a lot of great new features this year in
[C# 12](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12). My favorite is probably
[primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#primary-constructors) because of the
massive amount of boilerplate it replaces when writing classes that accept injected dependencies. I'm also super excited by the hidden
performance optimizations found in [collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#collection-expressions).

But, oddly, this year I'm going to talk about a new feature from last year. [Generic math](https://learn.microsoft.com/en-us/dotnet/standard/generics/math)
support was added in .NET 7 and is a great new feature by itself, but what's even more interesting to me is how it was implemented.
Under the hood, generic math support uses a new feature in C# 11 called [static virtual interface members](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/static-virtual-interface-members).

Static virtual interface members isn't a purely C# 11 feature. Utilizing them requires .NET 7 or later because changes
were also required in the .NET runtime. Since .NET 7 wasn't a long-term support release I don't think it's gotten
as much utilization as it deserves. But now .NET 8 is available and it is an LTS release, so I think it's worth revisiting.

Today, I'd like to demonstrate a different use case for static virtual interface members: generic type construction.

## The Backstory

Since .NET 2.0 we've had support for generics in C#. Generics are a great way to write code that can be reused across
a variety of types. For example, the `List<T>` class is a generic type which can be used to store a list of any distinct type
while maintaining strong type controls and avoiding boxing of value types (unlike its predecessor `ArrayList`).

Along with generics came the concept of [generic type constraints](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters).
These allow you to specify that a generic type must be a particular type or implement a particular interface. Since
the compiler then knows that the type will have certain members, it can allow you to use those members from within
the generic code.

One such constraint is the `new()` constraint. This requires that the type have a public parameterless constructor.
With this constraint, you can use the `new` keyword to construct an instance of the type. For example:

```cs
public abstract class Automobile
{
    public string Model { get; set; }
}

public class Car : Automobile
{
}

public class Motorcycle : Automobile
{
}

// This method may be called with either Car or Motorcycle to create
// a concrete instance of an Automobile.
public IEnumerable<T> Create<T>(IEnumerable<string> models)
    where T : Automobile, new()
{
    foreach (var model in models)
    {
        // Constructs T, either a Car or Motorcycle, depending on the
        // generic type parameter used when calling Create<T>
        yield return new T() { Model = model };
    }
}
```

## The Problem

This works great for simple cases, but what if we wanted to accept parameters on the constructor?
Perhaps we want to make the `Model` property read only and require it to be passed during construction.

{: .notice--info}
**Note:** I'm using C# 12 [primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#primary-constructors) here!

```cs
public abstract class Automobile(string model)
{
    public string Model { get; } = model;
}

public class Car(string model) : Automobile(model)
{
}

public class Motorcycle(string model) : Automobile(model)
{
}

// This method may be called with either Car or Motorcycle to create
// a concrete instance of an Automobile.
public IEnumerable<T> Create<T>(IEnumerable<string> models)
    where T : Automobile, new(string) // THIS IS NOT ALLOWED, COMPILER ERROR
{
    foreach (var model in models)
    {
        yield return new T(model); // THEREFORE, THIS IS NOT ALLOWED EITHER
    }
}
```

## Enter Static Virtual Interface Members

This is where static virtual interface members come in. With static virtual interface members, we can define a static
method on an interface that can be called from within the generic code. And, in many ways, a constructor is really a special
kind of static method. The differences between a constructor versus a static factory method that returns a new instance of `T`
are minor. So, let's see how we can use static virtual interface members to solve our problem.

```csharp
// This interface defines a static abstract method, meaning that any class which implements
// the interface must include a static method with the same signature. Note that generic T
// allows the Create method to return a strongly-typed instance.
public interface IAutomobileFactory<TSelf>
    // Constrain the generic type parameter to types that self-reference using IAutomobileFactory<TSelf>.
    // the Automobile constraint is optional in this example, but I like to include it for clarity.
    where TSelf : Automobile, IAutomobileFactory<TSelf>
{
    static abstract TSelf Create(string model);
}

public abstract class Automobile(string model)
{
    public string Model { get; } = model;
}

// IAutomobileFactory<Car> is included with a self-referencing generic type parameter
public class Car(string model) : Automobile(model), IAutomobileFactory<Car>
{
    // The create method is implemented as a static method on the class,
    // using Car explicitly as the return type.
    public static Car Create(string model) => new Car(model);
}

// Repeat for Motorcycle
public class Motorcycle(string model) : Automobile(model), IAutomobileFactory<Motorcycle>
{
    public static Motorcycle Create(string model) => new Motorcycle(model);
}

// This method may be called with either Car or Motorcycle to create
// a concrete instance of an Automobile.
public IEnumerable<T> Create<T>(IEnumerable<string> models)
    // Requires that T implement IAutomobileFactory<T> to gain access to the static method
    where T : Automobile, IAutomobileFactory<T>
{
    foreach (var model in models)
    {
        // Reference the static method via T
        yield return T.Create(model);
    }
}
```

## Conclusion

The main limitation of this approach is that you must be in control of the types of `T`. If `Car` and `Motorcycle`
were implemented in a third-party library then you may not be able to mark them with the `IAutomobileFactory<T>` interface.
Even if they implement the required static factory method, if it isn't marked with the interface then it won't work.
This is unlike the `new()` constraint which can be used with any type that has a public parameterless constructor.

The other limitation is static virtual interface members can only be used when targeting .NET 7 or later. This makes them unavailable
for legacy applications. It also limits their effectiveness for libraries shared via NuGet that include older TFMs like `netstandard2.0`.

However, when they're available to use, static virtual interface members can be a powerful tool for generic type construction that
I feel is underutilized. I hope this post has helped you understand how to apply them to your code.
