---
id: 314
title: String Interpolation Trickery and Magic with C# 10 and .NET 6
date: 2021-12-17T06:00:00-05:00
permalink: /csharp/2021/12/17/string-interpolation-trickery-and-magic-with-csharp-10-and-net-6
categories:
  - C#
  - .NET
summary: The .NET and C# teams have given as a great new performance tool with interpolated string handlers. But is there more to them than meets the eye?
---
{: .notice--info}
This blog is one of The December 17th entries on the [2021 C# Advent Calendar](https://www.csadvent.christmas/). Thanks for having me again Matt!

Recently, every November we've gotten a new version of C# paired with a new version of .NET. And, every year this new version is packed with
great new features. For me, one of the coolest features is [interpolated string handlers](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/).

Interpolated string handlers are primarily designed to provide a performance boost building strings. But is there more to them than meets the eye?
I believe that they lay the groundwork doing much more than just building strings faster.

## Interpolated String Handler Overview

First, let's start with an overview of how interpolated string handlers work. For a more in depth look, see the
[blog post from Stephen Toub](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/).

When using C# 9, interpolating a string is optimized by the compiler in a variety of ways. However, in many cases
the optimizations aren't an option, and a call to `string.Format(...)` is used. `string.Format` brings a lot of overhead,
such as interpreting the format string every call, potentially allocating an `object[]` on the heap, boxing value types,
and generating temporary intermediate strings.

For projects targeting .NET 6, even upgrading existing projects, string interpolations get an immediate performance
boost because they will use the [DefaultInterpolatedStringHandler](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.defaultinterpolatedstringhandler?view=net-6.0)
to build strings. This structure has a much better performance profile overall than `string.Format`.

```cs
// Example code, from Stephen Toub's post
public static string FormatVersion(int major, int minor, int build, int revision) =>
    $"{major}.{minor}.{build}.{revision}";
```

```cs
// Example equivalent code the compiler generates with .NET 6, from Stephen Toub's post
public static string FormatVersion(int major, int minor, int build, int revision)
{
    var handler = new DefaultInterpolatedStringHandler(literalLength: 3, formattedCount: 4);
    handler.AppendFormatted(major);
    handler.AppendLiteral(".");
    handler.AppendFormatted(minor);
    handler.AppendLiteral(".");
    handler.AppendFormatted(build);
    handler.AppendLiteral(".");
    handler.AppendFormatted(revision);
    return handler.ToStringAndClear();
}
```

## Custom Interpolated String Handlers

The `DefaultInterpolatedStringHandler` is really just the beginning, though. Methods which accept a string
may have an overload which accepts a custom interpolated string handler. When present, this causes C# to
use the custom handler you define rather than the default, allowing more advanced behaviors.

These behaviors can include stack allocations of scratch space, accepting other parameters to the method
call as constructor arguments to change behaviors, returning a boolean from the constructor that skips
the `AppendXXX` steps if we know they'll be unused, or short-circuiting additional `AppendXXX` steps if
we want to stop early.

A great example is `AssertInterpolatedStringHandler` which is available on `Debug.Assert` calls. It can
suppress most of the work building the string in the case where the first parameter to the Assert call
is `true`.

```cs
public void Example()
{
    var count = 0;

    // The interpolated string below will never be constructed in .NET 6, even when compiled in DEBUG mode
    Debug.Assert(count == 0, $"The count should be 0, but is {count}.");
}
```

The equivalent code for the above statement is something like:

```cs
public void Example()
{
    var count = 0;

    var condition = count == 0;
    var handler = new AssertInterpolatedStringHandler(31, 1, condition, out bool shouldAppend);
    if (shouldAppend)
    {
        handler.AppendLiteral("The count should be 0, but is ");
        handler.AppendFormatted(count);
        handler.AppendLiteral(".");
    }
    Debug.Assert(condition, handler);
}
```

If the `AppendXXX` methods return `bool` the result is checked after each call and can short-circuit part way through the operation.
For example, this might occur if the destination buffer runs out of space.

## Now Bring On The Magic

Now, I'd like you to take a moment and consider the `AssertInterpolatedStringHandler` example above. What does the example
have to do with strings? At what point are strings involved?

Wait for it...

The answer is "Only the literal segments are strings." Of course, `Debug.Assert` ends up making a string out of it by calling
`handler.ToStringAndClear();` But the `AssertInterpolatedStringHandler` is what's passed to `Debug.Assert`, not a string.
It can do whatever it likes with the handler. Additionally, the implication of `AppendFormatted` is that it will format `count`
as a string and append it. But in reality it may do whatever it likes.

Therefore, if we can imagine a "thing" which is built up of string literals and variable values presented in order, then
we can build it using an interpolated string. Even if what we're creating isn't a string at all.

## An Example: Parameterized SQL Queries

Have you ever needed to build a parameterized SQL query? Or [N1QL Query](https://www.couchbase.com/products/n1ql) if you're
a [Couchbase](https://www.couchbase.com/) user? It's very important to parameterize to help avoid injection attacks,
so the answer for most database users should be yes. Even if you use an ODM or ORM, it's often necessary to hand write queries
for special cases.

Building a parameterized query can be a pain. Take this relatively simple example for `SqlCommand`:

```cs
public async Task<string> GetOptionValue(int optionSet, string optionName)
{
    await using var cmd = new SqlCommand("SELECT OptionValue FROM Options WHERE OptionSet = @OptionSet AND OptionName = @OptionName", _connection);
    cmd.Parameters.Add("@OptionSet", SqlDbType.Int).Value = optionSet;
    cmd.Parameters.Add("@OptionName", SqlDbType.NVarChar).Value = optionName;

    return (await cmd.ExecuteScalarAsync()).ToString();
}
```

What if this could be written instead as follows, but still retained all the security of parameterized queries:

```cs
public async Task<string> GetOptionValue(int optionSet, string optionName)
{
    await using cmd = _connection.CreateCommand($"SELECT OptionValue FROM Options WHERE OptionSet = {optionSet} AND OptionName = {optionName}");

    return (await cmd.ExecuteScalarAsync()).ToString();
}
```

## Example Implementation

This example gets somewhat complicated, so I've tried to annotate it throughout with comments. First, the builder itself.

```cs
// This attribute let's C# know we're making an interpolated string handler
[InterpolatedStringHandler]
// The handler should usually be a "ref struct", meaning it only lives on the stack.
// This may be a limitation if you want to allow "await" within the holes in the expression, so "ref" may be removed in that case.
// However, this example requires "ref struct" because it includes a DefaultInterpolatedStringHandler in its fields.
public ref struct SqlCommandInterpolatedStringHandler
{
    // We'll use DefaultInterpolatedStringHandler to build the query string, it'll be more performant than reinventing the wheel.
    private DefaultInterpolatedStringHandler _innerHandler;

    // This will maintain a list of parameters as we build the query string
    public SqlParameter[] Parameters { get; }

    // The number of parameters added so far
    private int _parameterCount;

    public SqlCommandInterpolatedStringHandler(int literalLength, int formattedCount)
    {
        // Construct the inner handler, forwarding the same helper information
        _innerHandler = new DefaultInterpolatedStringHandler(literalLength, formattedCount);

        // Build an empty list of parameters with the capacity we'll need
        Parameters = new SqlParameter[formattedCount];
        _parameterCount = 0;
    }

    public void AppendLiteral(string value) =>
        // Forward literals to the inner handler to be added to the query string
        // In this example, literals represent query text like "SELECT ..."
        _innerHandler.AppendLiteral(value);

    public void AppendFormatted(ReadOnlySpan<char> value) =>
        // SqlParameters need strings not char spans, so forward to that implementation
        // Other backing implementations may be able to optimize this to avoid allocating a string
        AppendFormatted(value.ToString());

    public void AppendFormatted<T>(T value)
    {
        switch (value)
        {
            case int intValue:
                AppendParameter(SqlDbType.Int, intValue);
                break;

            case bool boolValue:
                AppendParameter(SqlDbType.Bit, boolValue);
                break;

            case string stringValue:
                AppendFormatted(stringValue);
                break;

            // Add support for more types here

            default:
                // Fallback for other types, we could make this smarter or throw an exception
                AppendFormatted(value?.ToString());
                break;
        }
    }

    // There are a lot of AppendFormatted overloads we're required to implement
    // We could also use alignment and format parameters for our own purposes, here we ignore them

    public void AppendFormatted<T>(T value, string? format) =>
        AppendFormatted(value);

    public void AppendFormatted<T>(T value, int alignment, string? format) =>
        AppendFormatted(value);

    public void AppendFormatted<T>(T value, int alignment) =>
        AppendFormatted(value);

    public void AppendFormatted(string? value) =>
        AppendParameter(SqlDbType.NVarChar, value);

    public void AppendFormatted(string? value, int alignment = 0, string? format = null) =>
        AppendParameter(SqlDbType.NVarChar, value);

    // Main handler for formatted segments
    private void AppendParameter(SqlDbType paramType, object? value)
    {
        // Since this is intended for use from compiler-generated code, we'll leave out typical runtime
        // preconditions like _parameterCount vs array length. We'll use Debug.Assert instead, and assume
        // the compiler used the type correctly for release builds.
        Debug.Assert(_parameterCount < Parameters.Length, "Exceeded formattedCount");

        // Create a unique parameter name, use an interpolated string builder with a stack-allocated buffer
        Span<char> paramNameBuffer = stackalloc char[8];
        var paramName = string.Create(null, paramNameBuffer, $"@Param{_parameterCount}");

        // Add the parameter name reference to the query string
        _innerHandler.AppendFormatted(paramName);

        // Add the parameter to the collection
        Parameters[_parameterCount] = new SqlParameter(paramName, paramType)
        {
            Value = value
        };
        _parameterCount++;
    }

    // Forward to the inner handler
    public readonly override string ToString() => _innerHandler.ToString();

    // Forward to the inner handler
    public string ToStringAndClear() => _innerHandler.ToStringAndClear();
}
```

This builder can then be invoked using this extension method:

```cs
public static class SqlConnectionExtensions
{
    public static SqlCommand CreateCommand(this SqlConnection connection,
        // The handler should be the last argument.
        // Where possible (i.e. non-async methods) it should usually be a by-ref argument.
        ref SqlCommandInterpolatedStringHandler handler)
    {
        // We must use ToStringAndClear(), not ToString(), to ensure we release resources
        var commandText = handler.ToStringAndClear();

        // Create the command and add the parameters stored in the handler
        var cmd = new SqlCommand(commandText, connection);
        cmd.Parameters.AddRange(handler.Parameters);
        return cmd;
    }
}
```

The generated SQL command text looks like this:

```sql
SELECT OptionValue FROM Options WHERE OptionSet = @Param0 AND OptionName = @Param1
```

It's that easy! Okay, maybe not quite easy, but still very powerful. Of course, there's also a
lot of room for improvement on this quick example.

- The parameters array could come from the array pool, though I'd want to measure that with benchmarks to be sure it's advantageous
- An overload that takes a stack-allocated `Span<SqlParameter>` (probably overkill given all the other heap allocations and boxing related to `SqlParameter`)
- A pre-built, static set of parameter names for reuse
- Support for more parameter types
- Using format strings to specify parameter types, i.e. varchar vs nvarchar
- An overload to create and execute the command rather than just create it, or perhaps create and prepare
- And probably much more I haven't considered

## Other Ideas

### Other Idea #1: JSON Building Without POCOs

This idea is interesting, but the double curly braces get a bit gnarly. Also, in most cases you probably
want a POCO, but I can see simple scenarios where POCOs are overkill. This example also assumes that the literals
are parsed at runtime to add double quotes around attribute names. This probably doesn't have great performance,
so it's more of a thought experiment, but who knows.

```cs
public string GetPersonJson(string name, int age, IEnumerable<Child> children)
{
    // name will get wrapped in quotes and special characters escaped, age would be a plain number,
    // children gets serialized as an array
    {% raw %}return JsonInterpolatedSerializer.Serialize($"{{name:{name},age:{age},children:{children}}}");{% endraw %}
}
```

### Other Idea #2: Safely Building HTML

In most cases, we build HTML in Razor views or pages. However, sometimes we need to build HTML in code.
This can be done with a TagBuilder, but that can feel unwieldy. Building a string that just applies
appropriate escaping would be nice. It could even use a TagBuilder under the hood. Though it probably wouldn't
be quite as performant given that the literals may need parsing.

```cs
public IHtmlContent CreateParagraph(string content, string class)
{
    // name will get wrapped in quotes and special characters escaped, age would be a plain number,
    // children gets serialized as an array
    return HtmlInterpolatedBuilder.Build($"<p class={class}>{content}</p>");
}
```

### Other Idea #3: Hash Codes

This is an example that doesn't involve strings at all. When overriding `object.Equals(object other)` its generally accepted that
you should also override `object.GetHashCode()`. However, calculating a hash code for your object may be cumbersome, especially
if there are a lot of fields. [System.HashCode](https://docs.microsoft.com/en-us/dotnet/api/system.hashcode?view=net-6.0) makes
this process easier, but still uses a lot of boilerplate.

```cs
public override int GetHashCode()
{
    // I know, I know, there's also HashCode.Combine<...>(...)
    // But that method can't override the comparer, and is limited to 8 fields.
    var hash = new HashCode();
    hash.Add(fieldA);
    hash.Add(fieldB);
    hash.Add(fieldC, StringComparer.OrdinalIgnoreCase);
    return hash.ToHashCode();
}
```

What if we had this syntax:

```cs
// In theory, if AppendLiteral is an inlined no-op JIT should optimize away the call
// This would drop any literal segments, such as whitespace, without perf penalty
// Just a theory, I haven't tested it
public override int GetHashCode() => HashCodeInterpolated.Calculate($"{fieldA} {fieldB} {fieldC:i}");
```

## Conclusion

In conclusion, I think the .NET community, especially library developers, should embrace the idea that interpolated string handlers
can be valuable for much more than just boosting string building performance. They open up a wide range of exciting new possibilities.

Some of my ideas may be silly in practice. I just wanted to get your creative juices flowing. If you have any other ideas,
I'd love to see them in the comments!
