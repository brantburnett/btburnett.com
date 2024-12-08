---
id: 316
title: Using vectorization in C# to boost performance
date: 2024-12-09T00:00:00-05:00
permalink: /csharp/2024/12/09/using-vectorization-in-csharp-to-boost-performance
categories:
  - C#
  - .NET
summary: "Vectorization, which is backed by SIMD instructinos, can significantly improve performance even when operating over relatively small datasets."
---
{: .notice--info}
This blog is one of The December 9th entries on the [2024 C# Advent Calendar](https://www.csadvent.christmas/). Thanks for having me again!

As an application developer, I rarely get to dig into low-level C# code. I feel like this is probably true for most of my fellow C# developers as well. We build our applications on top of the excellent work of the .NET team and others and watch performance improve each year as if by magic. However, every now and then I get my hands a bit dirtier, and it's a lot of fun.

For this year's C# Advent post, I decided to dust off an example of some low-level performance work I did last year. You'll actually find the changes in .NET starting with .NET 8, which I am inordinately proud of. It's just a minuscule little corner of the code that makes .NET tick, but I appreciate knowing I helped out just a tiny bit.

While you shouldn't need to implement this specific example, since it's baked into .NET 8 and later, I think there's a lot to learn from the process. So, without further ado, let me tell a ~~little~~ *long-winded* story about how it came to be.

## SIMD and Vectorization

Our story begins with an adhoc lesson on [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) to some of my coworkers. The concept of SIMD, which means Single Instruction Multiple Data, is to parallelize the same operation across multiple values simultaneously.

.NET represents SIMD via vector types. `Vector128<T>`, `Vector256<T>`, `Vector512<T>`, etc offer vectors of a particular size, which may or may not be hardware accelerated depending on CPU support. `Vector<T>` offers a vector of the "optimum" size for the current CPU. Optimum is a tricky beast to define, and can vary depending on needs, which is why I added bunny ears and why the specific-sized vectors are available. .NET generally implements `Vector<T>` as either 128-bit or 256-bit.

In my lesson, I took the example of adding a list of integers.

```c#
public int Sum(ReadOnlySpan<int> values)
{
    int accumulator = 0;
    foreach (var i in values)
    {
        accumulator += i;
    }

    return accumulator;
}
```

In this case, the CPU moves through all the values in the span, adding them one at a time to the accumulator. If this span contains 1024 integers, that's 1024 addition operations for the CPU to perform. There are also 1024 loop iterations which add even more cost to advance the loop and check the range to see if the end is reached.

Alternatively, a 128-bit vector can hold 4 32-bit integers. In the example of 1024 values in the span, if we can add 4 integers at a time that's only 256 addition operations and loop iterations instead of 1024. This is then followed by 3 final additions to add the 4 elements from the final vector. Obviously, this should be much faster.

## Imagine My Surprise

During this adhoc lesson, I decided to show off some code within .NET. The folks who work on the runtime are some of the smartest around, and I frequently refer to the .NET source to learn new things. I decided that showing the optimized code within LINQ for adding together numbers when they're known to be contiguous in memory would be a great example of SIMD optimization to show my coworkers.

So I shared my screen, pulled up the [.NET source](https://source.dot.net/), and this is what I found:

```c#
private static TResult Sum<T, TResult>(ReadOnlySpan<T> span)
    where T : struct, INumber<T>
    where TResult : struct, INumber<TResult>
{
    TResult sum = TResult.Zero;
    foreach (T value in span)
    {
        checked { sum += TResult.CreateChecked(value); }
    }

    return sum;
}
```

What? It's not using vectorization? The .NET team hasn't optimized this? My eyes lit up with glee, for in my hubris I thought I'd found an easy performance win! (Did you note my clever use of foreshadowing here?)

## Let's Do This!

I quickly threw together some vectorized code and did some performance testing. I don't have the results from that testing anymore, but it was greater than an order of magnitude faster on my Intel CPU on large arrays. The updated code looked something like this:

```c#
private static TResult Sum<T, TResult>(ReadOnlySpan<T> span)
    where T : struct, INumber<T>
    where TResult : struct, INumber<TResult>
{
    TResult sum = TResult.Zero;

    // Confirm that vectorization will work for T and TResult, will be accelerated
    // by SIMD instructions on the CPU, and we have enough elements to make it worthwhile.
    // All of these checks except the length check get elided during JIT compilation and don't
    // impact runtime performance. The math for the constant Vector<T>.Count * 2 is also folded
    // during JIT compilation. The end result for 128-bit vectors and a T of int is equivalent
    // to "if (span.Length >= 8)".
    if (typeof(T) == typeof(TResult)
        && Vector<T>.IsSupported
        && Vector.IsHardwareAccelerated
        && span.Length >= Vector<T>.Count * 2)
    {
        // Initialize the accumulator with all elements as zero
        Vector<T> accumulator = Vector<T>.Zero;
        do
        {
            // Load the next vector into a register from memory
            Vector<T> data = Vector.LoadUnsafe(ref MemoryMarshal.GetReference(span));

            // Add the data vector to the accumulator vector, which adds each of the elements 
            // independently using a single CPU operation
            accumulator += data;

            // Advance by the number of elements in the vector
            span = span.Slice(Vector<T>.Count);

        // Keep looping so long as we have at least one more vector's worth of data to process
        } while (span.Length >= Vector<T>.Count);

        // Sum the elements of the accumulator vector
        sum = TResult.CreateChecked(Vector.Sum(accumulator));

        // Fall through to the loop below to pick up any remaining elements in the span
        // in case it wasn't a precise multiple of Vector<T>.Count elements
    }

    foreach (T value in span)
    {
        checked { sum += TResult.CreateChecked(value); }
    }

    return sum;
}
```

## Not So Fast, Smartypants

At this point, however, my brain started to kick back into gear. This was too easy. I couldn't imagine a world in which the brilliant minds working on the .NET team hadn't already vectorized this code. As supporting evidence, the LINQ code for average, min, and max was already vectorized. There had to be some detail I was missing.

My eyes locked onto this key line in the original method:

```c#
checked { sum += TResult.CreateChecked(value); }
```

The original code was explictly enabling overflow checking using the `checked` keyword. This causes an exception to be thrown if the value of the accumulator overflows the size of the integer type. This is done relatively inexpensively using flags on the CPU. When the CPU completes an add operation, it knows if there was an overflow or not and sets or clears the overflow flag. .NET emits a conditional jump operation (such as `jo` Jump if Overflow on x64) that branches to exception logic if that flag is set, so the only cost is the extra jump operation.

A bit of quick Googling yielded the answer. For [Intel x86/x64](https://www.felixcloutier.com/x86/paddb:paddw:paddd:paddq): `When an individual result is too large to be represented in 32 bits (overflow), the result is wrapped around and the low 32 bits are written to the destination operand (that is, the carry is ignored)`. ARM NEON behaves similarly. This certainly explains why the .NET team hadn't implemented vectorization for `Sum`. Overflows are just completely ignored by the CPU and thus my new code, so it isn't actually equivalent because it fails to throw exceptions.

## Climbing Out of the Pit of Despair

Unfortunately (or perhaps fortunately), sometimes my brain just gets stuck on something and can't let it go. After I got done crying in my beer, I just couldn't accept the idea that there wasn't some way to make this work. I decided to try to implement my own manual overflow checking since the CPU wasn't helping.

This is where things start to get fun, we get to do some bit-twiddling magic! First, a refresher on how signed integers are stored using [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement). Among other details, the most significant bit (bit 31 in the case of 32-bit integers) is the sign bit. If this bit is 1, we have a negative number, if it is 0 then the number is positive.

We can use this bit to help detect overflows. If we're adding a negative and a positive number together, an overflow isn't possible. By definition, the result will be somewhere between the two values and will therefore fit in the bits available. We only need to worry if the both numbers are negative (both sign bits are 1) or both numbers are positive (both sign bits are 0). If we add two positive numbers, then the sign bit should stay 0 (still positive). If we add two negative numbers, then the sign bit should stay 1 (still negative). But if there's an overflow, we're guaranteed that the sign bit will change (overflowing negative produces a positive number, and vice versa). It isn't possible, in a single addition, to wrap so far you get back to the same sign again.

Therefore, we can test for an overflow after each addition by looking for the case where A) both sign bits are the same before adding and B) the sign bit is different after adding.

```c#
int sum = x + y;

// The exclusive ors test for a change in the sign bit between the each input and the sum.
// The and operation then ensures we only care if BOTH are different. If only one is different,
// we were adding a positive and negative number together, which can't overflow.
// int.MinValue has only the sign bit set, all other bits are 0, so the and operation will then
// mask out the non-sign bits. If any bit is still set (!= 0) we've overflowed.
bool isOverflow = (sum ^ x) & (sum ^ y) & int.MinValue != 0;
```

Great news, all of these bit-twiddling calls such as `&` and `^` are also available as SIMD instructions, so we can vectorize the overflow checking as well.

```c#
// Build a test vector with only the sign bit set in each element.
Vector<int> overflowTestVector = new(int.MinValue);

// Process vectors
Vector<int> accumulator = Vector<int>.Zero;
do
{
    Vector<int> data = Vector.LoadUnsafe(ref MemoryMarshal.GetReference(span));
    Vector<int> temp = accumulator + data;

    Vector<int> overflowCheck = (temp ^ accumulator) & (temp ^ data);

    // Mask out everything except the sign bit and see if it's set on any element in the vector
    if ((overflowTracking & overflowTestVector) != Vector<int>.Zero)
    {
        ThrowHelper.ThrowOverflowException();
    }

    accumulator = temp;
    span = span.Slice(Vector<int>.Count);
} while (span.Length >= Vector<int>.Count);
```

## Almost There...

The next step after this was performance testing. To my chagrin, at first this new algorithm was actually slower than the original unvectorized approach. The reason is somewhat obvious, the extra work required to manually check for overflows is more expensive than the savings from vectorization. For each vector in the loop, overflow checking is roughly adding the following extra operations:

- Two exclusive or operations
- Two and operations
- One comparison operation with a cold branch
- One assignment operation to store `temp` in `accumulator`

That said, while it was slower, it was only a little slower. A bit more work and we can get there. In the interests of not boring everyone to tears I'll simply summarize the final adjustments that I found after lots of experimentation and hair pulling.

One change was to switch to using a `ref T` variable instead of `ReadOnlySpan<T>` within the loop. This is slightly riskier because now the framework isn't doing range checking for buffer overflows, our code must do it correctly instead. But it turns out to be worthwhile here.

The most valuable change was using [loop unrolling](https://en.wikipedia.org/wiki/Loop_unrolling). I won't dig into the concept too deeply here, but the idea is to process more than one vector per loop. In this case, the loop is rewritten to process four vectors every time instead of one. This allows us to completely eliminate the assignment of `temp` back to `accumlator`. Instead, we alternate accumulators every other vector, which works so long as we have an even number of vectors in the loop. We can also check for overflows every four vectors instead of after each one, meaning fewer branches.

```c#
nuint index = 0;
nuint limit = length - (nuint)Vector<T>.Count * 4;
do
{
    Vector<T> data = Vector.LoadUnsafe(ref ptr, index);
    Vector<T> accumulator2 = accumulator + data;
    Vector<T> overflowTracking = (accumulator2 ^ accumulator) & (accumulator2 ^ data);

    data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count);
    accumulator = accumulator2 + data;
    overflowTracking |= (accumulator ^ accumulator2) & (accumulator ^ data);

    data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count * 2);
    accumulator2 = accumulator + data;
    overflowTracking |= (accumulator2 ^ accumulator) & (accumulator2 ^ data);

    data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count * 3);
    accumulator = accumulator2 + data;
    overflowTracking |= (accumulator ^ accumulator2) & (accumulator ^ data);

    if ((overflowTracking & overflowTestVector) != Vector<T>.Zero)
    {
        ThrowHelper.ThrowOverflowException();
    }

    index += (nuint)Vector<T>.Count * 4;
} while (index < limit);
```

This change did require increasing the minimum data length required to perform vectorization from
two vectors to four vectors. Additionally, I added the requirement `Vector<T>.Count > 2`. On some CPUs, especially ARM,
`Vector<T>` may be 128 bits. This means they can only hold two 64-bit `long` elements per vector.
From a savings perspective, all of the overflow checking effort simply isn't worthwhile to add two
numbers at a time instead of one. So the vectored path only applies to `long` elements if the vector
is at least 256 bits.

## The Final Product

The [final PR](https://github.com/dotnet/runtime/pull/84519) has all of the details, but here is a snippet of the final product
with some slightly different comments inline.

```c#
private static TResult Sum<T, TResult>(ReadOnlySpan<T> span)
    where T : struct, INumber<T>
    where TResult : struct, INumber<TResult>
{
    if (typeof(T) == typeof(TResult)
        && Vector<T>.IsSupported
        && Vector.IsHardwareAccelerated
        && Vector<T>.Count > 2
        && span.Length >= Vector<T>.Count * 4)
    {
        // These magic branches are elided by JIT, they are required because
        // SumSignedIntegersVectorized adds the IBinaryInteger<T>, ISignedNumber<T>, 
        // and IMinMaxValue<T> type constraints. They don't have any performance
        // impact at runtime.
        if (typeof(T) == typeof(long))
        {
            return (TResult) (object) SumSignedIntegersVectorized(MemoryMarshal.Cast<T, long>(span));
        }
        if (typeof(T) == typeof(int))
        {
            return (TResult) (object) SumSignedIntegersVectorized(MemoryMarshal.Cast<T, int>(span));
        }
    }

    TResult sum = TResult.Zero;
    foreach (T value in span)
    {
        checked { sum += TResult.CreateChecked(value); }
    }

    return sum;
}

private static T SumSignedIntegersVectorized<T>(ReadOnlySpan<T> span)
    where T : struct, IBinaryInteger<T>, ISignedNumber<T>, IMinMaxValue<T>
{
    Debug.Assert(span.Length >= Vector<T>.Count * 4);
    Debug.Assert(Vector<T>.Count > 2);
    Debug.Assert(Vector.IsHardwareAccelerated);

    ref T ptr = ref MemoryMarshal.GetReference(span);
    nuint length = (nuint)span.Length;

    Vector<T> accumulator = Vector<T>.Zero;

    // Build a test vector with only the sign bit set in each element.
    Vector<T> overflowTestVector = new(T.MinValue);

    // Unroll the loop to sum 4 vectors per iteration. This reduces range check
    // and overflow check frequency, allows us to eliminate move operations swapping
    // accumulators, and may have pipelining benefits.
    nuint index = 0;
    nuint limit = length - (nuint)Vector<T>.Count * 4;
    do
    {
        // Switch accumulators with each step to avoid an additional move operation
        Vector<T> data = Vector.LoadUnsafe(ref ptr, index);
        Vector<T> accumulator2 = accumulator + data;
        Vector<T> overflowTracking = (accumulator2 ^ accumulator) & (accumulator2 ^ data);

        data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count);
        accumulator = accumulator2 + data;
        overflowTracking |= (accumulator ^ accumulator2) & (accumulator ^ data);

        data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count * 2);
        accumulator2 = accumulator + data;
        overflowTracking |= (accumulator2 ^ accumulator) & (accumulator2 ^ data);

        data = Vector.LoadUnsafe(ref ptr, index + (nuint)Vector<T>.Count * 3);
        accumulator = accumulator2 + data;
        overflowTracking |= (accumulator ^ accumulator2) & (accumulator ^ data);

        if ((overflowTracking & overflowTestVector) != Vector<T>.Zero)
        {
            ThrowHelper.ThrowOverflowException();
        }

        index += (nuint)Vector<T>.Count * 4;
    } while (index < limit);

    // Process remaining vectors, if any, without unrolling
    limit = length - (nuint)Vector<T>.Count;
    if (index < limit)
    {
        Vector<T> overflowTracking = Vector<T>.Zero;

        do
        {
            Vector<T> data = Vector.LoadUnsafe(ref ptr, index);
            Vector<T> accumulator2 = accumulator + data;
            overflowTracking |= (accumulator2 ^ accumulator) & (accumulator2 ^ data);
            accumulator = accumulator2;

            index += (nuint)Vector<T>.Count;
        } while (index < limit);

        if ((overflowTracking & overflowTestVector) != Vector<T>.Zero)
        {
            ThrowHelper.ThrowOverflowException();
        }
    }

    // Add the elements in the vector horizontally.
    // Vector.Sum doesn't perform overflow checking, instead add elements individually.
    T result = T.Zero;
    for (int i = 0; i < Vector<T>.Count; i++)
    {
        checked { result += accumulator[i]; }
    }

    // Add any remaining elements
    while (index < length)
    {
        checked { result += Unsafe.Add(ref ptr, index); }

        index++;
    }

    return result;
}
```

## Conclusion

What was the final result? Well, if I'm being honest, it was me bouncing around the house giggling like a school boy because my change got mentioned by the inestimable [Stephen Toub](https://github.com/stephentoub) in his annual blog post on [Performance Improvements in .NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/). Getting a minor mention was a goal I'd set for myself for the year, and one I honestly didn't think I'd accomplish.

Here are the performance metrics he showed in his post for adding 1024 32-bit integers, a 77% improvement!

| Method | Runtime  | Mean      | Ratio |
| Sum    | .NET 7.0 | 347.28 ns | 1.00  |
| Sum    | .NET 8.0 | 78.26 ns  | 0.23  |

Hopefully this journey as a reader was as fun for you as it was for me as an author. I know I learned a lot implementing it, and I really encourage everyone to poke around in the .NET source. You never know what you might learn or, even better, what you might find that you can contribute.