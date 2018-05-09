---
id: 274
title: 'Who Says C# Interfaces Can''t Have Implementations?'
date: 2017-12-07T09:00:29+00:00
guid: http://btburnett.com/?p=274
permalink: /2017/12/who-says-c-interfaces-cant-have-implementations.html
categories:
  - C#
---
_This post is the seventh installment of the [2017 C# Advent Calendar](https://crosscuttingconcerns.com/The-First-C-Advent-Calendar) operated by [Matthew Groves](https://crosscuttingconcerns.com/). Thanks for letting me participate!_

## Overview

According to the [C# Programming Guide](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/interfaces/), an interface cannot contain an implementation:

> When a class or struct implements an interface, the class or struct must provide an implementation for all of the members that the interface defines. The interface itself provides no functionality that a class or struct can inherit in the way that it can inherit base class functionality.

There is a [new feature in the works for C# 8](https://www.infoq.com/news/2017/04/Clr-Default-Methods) that will provide default interface implementations. However, I'm known for my impatience, and I don't want to wait. The fact is, interfaces **can** have implementations in C# today.

**Note:** At this point, I'd like to give a shout out to the ASP.NET Core team. I can't really take credit for this concept, I learned it by digging through the [ASP.NET Core source on GitHub](https://github.com/aspnet).
{: .notice--info}

## The Magic of Extension Methods

When I first encountered extension methods, I viewed them as a helpful way to add functionality to third-party classes. Adding additional logic to System.DateTime is helpful, right? However, another powerful use case is to create extensions for your own interfaces. When you write an extension method for an interface, you're actually providing it with implementation. Sure, the consumer may need an extra using statement at the top of the file, but it's still an implementation! Taking this approach has several advantages:

* Extension methods can be safely added to interfaces without risking backward compatibility for consumers (especially if they're in a separate namespace).
* The interface becomes easier to implement there are few methods on the interface itself which must be implemented.
* There is guaranteed consistency in method implementation across all class implementations.

There is, however, one rule you must follow. The extension method must only use members exposed by the interface being extended. While hacks like typecasting back to a particular implementation will work, it risks the purity of the implementation. What happens when the extension is used against a different implementation of the interface? Generally speaking, this rule should be broken only for performance optimizations and should have a fallback that works against any implementation.

## The Elephant In The Room

Perhaps the best-known example of extension methods in .NET is [LINQ](https://msdn.microsoft.com/en-us/library/bb308959.aspx?f=255&MSPPError=-2147217396). The vast majority of the API surface of LINQ is extension methods on the IEnumerable<T> and IQueryable<T> interfaces.

LINQ was added in .NET 3.0. If the LINQ team had decided to add the LINQ methods directly to IEnumerable<T>, this would have constituted a breaking change. Every class which implemented IEnumerable<T> would have been broken upon upgrading to .NET 3.0 until the _entire suite of LINQ methods was implemented._ Just imagine implementing that across every collection class. C# developers everywhere would have chased the LINQ team around with pitchforks.

Instead, the LINQ team implemented their system as extension methods. The result is that .NET 3.0 and LINQ were fully backward compatible with libraries written in .NET 2.0. Additionally, a whole lot of headaches and duplicated code were avoided. And it was done by, effectively, adding implementations to interfaces.

# So, Teach Me The Magic

Writing extension methods is actually relatively painless. They're basically just syntactic sugar layered on top of static methods.

Let's start with this interface:

```cs
namespace MyNamespace
{
    public interface IMyInterface
    {
        IList&lt;int&gt; Values { get; set; }
    }
}
```

Now, let's add an extension method that returns the count of all values in the list greater than a threshold.

```cs
namespace MyNamespace
{
    public static class MyInterfaceExtensions
    {
        public static int CountGreaterThan(this IMyInterface myInterface, int threshold)
        {
            return Values?.Where(p =&gt; p &gt; threshold).Count() ?? 0;
        }
    }
}
```

The extension method can be consumed like this:

```cs
using MyNamespace;

// ...

public void DoSomething()
{
    var myImplementation = new MyInterfaceImplementation();

    // Note that there's no typecast to IMyInterface required
    var countGreaterThanFive = myImplementation.CountGreaterThan(5);
}

// ...
```

There are four key pieces to the puzzle:

1. MyInterfaceExtensions and CountGreaterThan are both public (though they could be internal if you want to use them only within your library).
2. MyInterfaceExtensions and CountGreaterThan are both static.
3. The first parameter of CountGreaterThan is the interface and is preceded by the "this" keyword.
4. The file where DoSomething is declared includes a using statement for the namespace where the extensions are declared (Visual Studio will help by adding this automatically).

**Note:** There are many different approaches for code organization surrounding these extensions methods. Some teams may prefer them in the same file, others in a separate file in the same folder, and others may want extensions in a separate folder/namespace. Just be sure your team picks a pattern and sticks with it. For teams that choose separate files, including comments on the interface that point to the extension files is a good idea.
{: .notice--info}

## Making Mocks Easy, One Extension Method At A Time

My favorite use case is for supporting unit tests, especially when I'm providing lots of method overloads. The reason I point out this specific use case it to show that extension methods can be very useful outside of developing reusable libraries. Almost all development today requires unit testing.

When writing unit tests, it's often best to test against mocks of interfaces, rather than real implementations. The reasons why are beyond the scope of this post, so just trust me on this. I used to think it was silly until I learned the hard way it wasn't. Creating mocks can be greatly simplified by adding implementation to interfaces.

For example, imagine an interface named ICartItem which represents an item in an online shopping cart. It needs several different methods to support changing the quantity in the cart.

```cs
public interface ICartItem
{
    int Quantity { get; set; }
    void IncrementQuantity();
    void IncrementQuantity(int delta);
    void DecrementQuantity();
}
```

The Quantity property can be set directly if the user enters a value, but there are also up and down arrows on the UI which tie to the IncrementQuantity and DecrementQuantity methods.

Continuing the example, ICartItem is used by a CartManager class that manages the shopping cart, and CartManager contains lots of business logic that requires unit tests. These tests require mock implementations of ICartItem. Since the interface defines a property and three methods, every mock needs to include four implementations. However, the three methods have a very basic function, and their implementation should be consistent across all classes that implement the interface. Additionally, they can be written to be supported by the Quantity property.

```cs
public interface ICartItem
{
    int Quantity { get; set; }
}

public static class CartItemExtensions
{
    public static void IncrementQuantity(this ICartItem cartItem)
    {
        cartItem.IncrementQuantity(1);
    }

    public static void IncrementQuantity(this ICartItem cartItem, int delta)
    {
        cartItem.Quantity += delta;
    }

    public static void DecrementQuantity(this ICartItem cartItem)
    {
        cartItem.IncrementQuantity(-1);
    }
}

public class CartManagerTests
{
    [Fact]
    public void Some_Test()
    {
        // Arrange

        // Note: This example uses Moq (https://www.nuget.org/packages/Moq/), your syntax may vary
        var mockCartItem = new Mock&lt;ICartItem&gt;();
        mockCartItem.SetupAllProperties();

        // ...
    }
}
```

Now every mock of ICartItem is greatly simplified, and they will have support for the IncrementQuantity and DecrementQuantity methods without any specific code added to the tests.

## More Advanced, Real-world Examples

There are lots of great examples of this pattern found throughout the ASP.NET Core source code on GitHub. Here is a small selection:

* [ConsoleLoggerFactoryExtensions](https://github.com/aspnet/Logging/blob/rel/1.1.2/src/Microsoft.Extensions.Logging.Console/ConsoleLoggerFactoryExtensions.cs) &#8211; Note how each extension implements overloads by forwarding to other extensions with more detailed parameters, until finally a call to ILoggerFactory.AddLogger(ILoggerProvider) is reached.
* [ServiceCollectionServiceExtensions](https://github.com/aspnet/DependencyInjection/blob/rel/1.1.1/src/Microsoft.Extensions.DependencyInjection.Abstractions/ServiceCollectionServiceExtensions.cs) &#8211; A veritable avalanche of extensions, which eventually call the most powerful method, IServiceCollection.Add(ServiceDescriptor).
* [ResponseCachingExtensions](https://github.com/aspnet/ResponseCaching/blob/rel/1.1.2/src/Microsoft.AspNetCore.ResponseCaching/ResponseCachingExtensions.cs) &#8211; In this case, a descendent assembly rather than the original assembly adds an extension to the interface. This helps reduce the inclusion of unnecessary dependencies, users only add the packages they need to the application. Yet, ease-of-use is maintained by extending the original interface.

## Summary

That's how to add implementations to interfaces in a nutshell. It's a very powerful tool but is especially useful for writing shared libraries as well as for writing code that is easy to unit test. Key use cases to watch out for include:

* Overloads where simpler methods are just forwarding the requests to methods with more parameters and filling in default values. The simpler methods can be extensions.
* Helper methods that support more common uses cases by forwarding method calls to more powerful but less frequently used methods.
* Other methods where the implementation will always be the same for every class that implements the interface, and which are supported by other members of the interface.
