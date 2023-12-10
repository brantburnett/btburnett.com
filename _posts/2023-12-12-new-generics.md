---
id: 316
title: Generic Type Construction With Static Virtual Interface Members
date: 2022-12-12T00:00:00-05:00
permalink: /csharp/2023/12/12/generic-type-construction-with-static-virtual-interface-members
categories:
  - C#
  - .NET
summary: One day I asked myself "Hey, I wonder if I can use Roslyn to make an even better, faster OpenAPI SDK generator?"
---
{: .notice--info}
This blog is one of The December 12th entries on the [2023 C# Advent Calendar](https://www.csadvent.christmas/). Thanks for having me again!

Typically for C# Advent I write about a new C# feature. And there are a lot of great new features this year in
[C# 12](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12). My favorite is probably
[primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#primary-constructors) because of the
massive amount of boilerplate it removes from my code when I write services that accept injected dependencies. And my love
of high performance code is also super excited by the hidden optimizations found in
[collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#collection-expressions).

But, oddly, this year I'm going to talk about a new feature from last year in .NET 7. The [generic math](https://learn.microsoft.com/en-us/dotnet/standard/generics/math)
support that was added in .NET 7 is a great new feature by itself, but what's even more interesting to me is how it was implemented.
Under the hood, generic math support uses a new feature in C# 11 called [static virtual interface members](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/static-virtual-interface-members).

Station virtual interface members isn't a purely C# 11 feature. Utilizing them requires .NET 7 or later because backing
support was already required in the runtime. Since .NET 7 wasn't a long-term support release I don't think it's gotten
as much utilization as it deserves.

Today, I'd like to demonstrate a different use case for static virtual interface members: generic type construction.

## The Backstory

Since .NET 2.0 we've had support for generics in C#. Generics are a great way to write code that can be reused across
a variety of types. For example, the `List<T>` class is a generic type which can be used to store a list of any type
while maintaining strong type controls and avoiding boxing/unboxing of value types.

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
        yield return new T() { Model = model };
    }
}
```

## The Problem

This works great for simple cases, but what if we wanted to accept parameters on the constructor?
Perhaps we want to make the `Model` property read only and require it to be passed in on the constructor.

{: .notice--info}
Note: I'm using C# 12 [primary constructors](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#primary-constructors) here!

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
        yield return new T(model);
    }
}
```

## Enter Static Virtual Interface Members

This is where static virtual interface members come in. With static virtual interface members, we can define a static
method on an interface that can be called from within the generic code. And, in many ways, a constructor is really a special
kind of static method. The differences between a constructor versus a static method that returns a new instance of `T`
are minor. So, let's see how we can use static virtual interface members to solve our problem.

```csharp
// This interface defines a static abstract method, meaning that any class which implements
// the interface must include a static method with the same signature. Note that generic T
// allows the Create method to return a strongly-typed instance.
public interface IAutomobileFactory<T>
  where T : Automobile // This constraint is optional, but I like to include it for clarity.
{
    static abstract T Create(string model);
}

public abstract class Automobile(string model)
{
    public string Model { get; } = model;
}

// IAutomobileFactory<T> is included with a self-referencing generic type parameter
public class Car(string model) : Automobile(model), IAutomobileFactory<Car>
{
    // The create method is implemented as a static method on the class,
    // using Car explicitly as the return type.
    public static Car Create(string model) => new Car(model);
}

public class Motorcycle(string model) : Automobile(model), IAutomobileFactory<Motorcycle>
{
    public static Motorcycle Create(string model) => new Motorcycle(model);
}

// This method may be called with either Car or Motorcycle to create
// a concrete instance of an Automobile.
public IEnumerable<T> Create<T>(IEnumerable<string> models)
    // Require that T implement IAutomobileFactory<T> to gain access to the static method
    where T : Automobile, IAutomobileFactory<T>
{
    foreach (var model in models)
    {
        // Reference the static method via T
        yield return T.Create(model);
    }
}
```

## Limitations

The main limitation of this approach is that you must be in control of the types of T. A class from a third-party library may implement the required static factory method, but if it isn't marked with the `IAutomobileFactory<T>` interface then you won't be able to use it. This is unlike the `new()` constraint which can be used with any type that has a public parameterless constructor.

The other limitation is that static virtual interface members can only be used when targeting .NET 7 or later. This makes it unavailable for legacy applications and limits its effectiveness for libraries shared via NuGet that include older TFMs like .NET Standard 2.0.

## Conclusion

