---
layout: post
title: "High-performance string formatting in .NET"
image: /assets/img/2025-05-14-high-performance-strings-preview.png
date: 2025-05-14 14:00:00 +0200
---
One of the topics I covered in my previous post on
[memory optimizations]({{ site.baseurl }}{% post_url 2025-04-10-memory-optimizations %}) was string formatting.
As I was writing that post, I uncovered an embarrassing gap in my knowledge!

In .NET Framework, the default string formatting method was `string.Format`, which requires the runtime to
call `ToString` on all its arguments. For example, `string.Format("{0} = {1}", Key, Value)` would internally call
both `Key.ToString()` and `Value.ToString()` to produce the final result. These two extra allocations
introduce a performance penalty, which is unfortunate because we don't need those temporary
strings—we only need the final formatted result.

Having worked with the .NET Framework for quite a long time, I had assumed that interpolated strings like
`$"{Key} = {Value}"` worked the same way. I was excited to learn that I was wrong! .NET Core has come a long
way in eliminating unnecessary allocations and boxing. In today's blog post, we'll explore high-performance
`ToString` alternatives that help you avoid temporary string allocations, and in some cases, avoid allocations
altogether!

## How string interpolation works

Before we dive into high-performance topics, let's refresh our memory on how string interpolation works behind
the scenes. Imagine you are working with the following classes:

```csharp
public record Point3D(double X, double Y, double Z)
{
    public override ToString() => $"({X}, {Y}, {Z})";
}

public record Line3D(Point3D Start, Point3D End)
{
    public override ToString() => $"[{Start}, {End}]";
}
```

If you inspect the generated code for the `Line3D` class with a .NET decompiler such as
[ILSpy](https://github.com/icsharpcode/ILSpy), you will see that all the heavy lifting in its
`ToString` implementation is done at compile time. The compiler transforms the interpolation
expression into a sequence of `AppendLiteral` and `AppendFormatted` calls:

```csharp
var handler = new DefaultInterpolatedStringHandler(4, 2);

handler.AppendLiteral("[");
handler.AppendFormatted(Start);
handler.AppendLiteral(", ");
handler.AppendFormatted(End);
handler.AppendLiteral("]");

return handler.ToStringAndClear();
```

The code is self-explanatory. Literal parts of the interpolation expression are added to the interpolated string
handler using `AppendLiteral` calls, while placeholder values are added using `AppendFormatted` calls. The handler
internally uses a temporary `char` buffer from the `ArrayPool<char>.Shared` and allocates memory only when the
final string is created.

The formatting of placeholder values happens in the `AppendFormatted` method. If the object you want to format
overrides only `ToString`, then the runtime must call `ToString`. This will create a temporary string, wasting
both CPU and memory (remember, we only need the final interpolated result). In the case of `Line3D`, two unnecessary
strings would be allocated (one for each point). But if a type implements the `ISpanFormattable` interface,
something magical will happen.

## ISpanFormattable

Without much fanfare, .NET 6 introduced an important interface called
[ISpanFormattable](https://learn.microsoft.com/en-us/dotnet/api/system.ispanformattable):

```csharp
public interface ISpanFormattable : IFormattable
{
    public bool TryFormat(
        Span<char> destination,
        out int charsWritten,
        ReadOnlySpan<char> format,
        IFormatProvider? provider
    );
}
```

The interface serves a simple purpose: when a placeholder in an interpolated string needs to be filled,
`DefaultInterpolatedStringHandler` checks whether the type implements `ISpanFormattable`. If it does, the
`TryFormat` method is used to write the content directly into the final buffer, bypassing `ToString` entirely.

Types such as `int` and `DateTime` have implemented this interface from day one, so when you write something like
`$"Today is {DateTime.Now}"`, a single allocation occurs—the one for the resulting string.

Of course, the use of this interface is not limited to primitive types. You can implement `ISpanFormattable` in any
type, and your `TryFormat` implementation will be called instead of `ToString` during string interpolation. Although
the signature of `TryFormat` might look intimidating, implementing it is usually quite straightforward.

## Implementing ISpanFormattable

The official docs don't explain how to implement `ISpanFormattable`, and almost all online blog posts showcase
overly complicated implementations. Let me try to fix that.

First, let's talk about how *not* to implement the interface. Since you have access to the destination buffer,
you could manually write every component of your type's string representation into it. You would also need to
perform bounds checks, but all of this would make your code very fragile and error-prone.

Fortunately, there is a much simpler way to implement `TryFormat`: by calling the `MemoryExtensions.TryWrite`
extension method and passing it the exact same string format you would use in a `ToString` implementation.
The easiest way to explain how this method works is simply to show how it could be used in `Point3D.TryFormat`:

```csharp
public bool TryFormat(
    Span<char> destination,
    out int charsWritten,
    ReadOnlySpan<char> format,
    IFormatProvider provider) =>
        destination.TryWrite(provider, $"({X}, {Y}, {Z})", out charsWritten);
```

That's literally all! You might be thinking that the `$"({X}, {Y}, {Z})"` expression generates a temporary
string, but that's not the case. If you look closely, you'll notice that the type of that function argument
is not `string`, but `MemoryExtensions.TryWriteInterpolatedStringHandler`. This means the compiler is doing
the same heavy lifting as before—transforming the string template into a sequence of `AppendLiteral` and
`AppendFormatted` calls that write directly into `destination`. I won't go into detail, but if you are
interested in how the compiler does this, you can find the explanation
[here](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/).

Now let's measure the performance improvements using [BenchmarkDotNet](https://benchmarkdotnet.org/). Here
are the results of a benchmark comparing two versions of `$"{line}"` interpolation—one where `Line3D`
implements only `ToString`, and one where it also implements `TryFormat`:

| Method    | Mean     | Error   | StdDev  | Ratio | Allocated | Alloc Ratio |
|---------- |---------:|--------:|--------:|------:|----------:|------------:|
| ToString  | 620.4 ns | 8.36 ns | 7.82 ns |  1.00 |     336 B |        1.00 |
| TryFormat | 433.2 ns | 4.00 ns | 3.74 ns |  0.70 |     104 B |        0.31 |

`ISpanFormattable` is a clear winner in both execution speed (30% improvement in the `Ratio` column) and
memory usage (69% improvement in the `Alloc Ratio` column). And the greatest thing about this optimization
is that it basically didn't cost us anything—we used the same string format we previously used in `ToString`!

If you are worried about duplicating the string format in the `ToString` and `TryFormat` methods, there is
an elegant solution to that as well. You can implement `ToString` like this and rely on the fact that the string
interpolation will internally call `TryFormat`, producing the same result:

```csharp
public override string ToString() => $"{this}";
```

Isn't that neat? You gotta love modern .NET! But the story of high-performance string formatting doesn't end here.

## UTF-8 string literals

In .NET, strings are stored in memory using UTF-16 encoding. But what happens when you need to send a string over
the network or save it to a file? The standard encoding for these purposes is UTF-8, so in most cases you need to
transcode your strings to UTF-8 first. Wouldn't it be nice if you could encode strings directly as UTF-8 bytes,
avoiding the unnecessary conversion?

Ever since C# 11, you have been able to create
[UTF-8 string literals](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-11.0/utf8-string-literals)
as `ReadOnlySpan<byte>` objects by adding the `u8` suffix to string constants:

```csharp
ReadOnlySpan<byte> hello = "Hi there, hello!"u8;
```

This syntax is great for *constant* strings, but it doesn't allow us to use string interpolation, which significantly
limits its usefulness. But don't worry—you wouldn't be reading this post if the .NET team didn't have a solution for
that too. Say hello to one of the latest additions to .NET:
[IUtf8SpanFormattable](https://learn.microsoft.com/en-us/dotnet/api/system.iutf8spanformattable).

## IUtf8SpanFormattable

Let me show you the interface first, and you can try to guess what it's used for:

```csharp
public interface IUtf8SpanFormattable
{
    bool TryFormat(
        Span<byte> destination,
        out int bytesWritten,
        ReadOnlySpan<char> format,
        IFormatProvider? provider
    );
}
```

If your first thought was that it looks very similar to
`ISpanFormattable`, you were absolutely right—these two interfaces are almost identical.
The key difference is that `IUtf8SpanFormattable` is used to format strings into UTF-8
byte buffers, so the destination must be a `Span<byte>` instead of a `Span<char>`.

Similar to `ISpanFormattable`, this new interface can be easily implemented using regular
string interpolation syntax. Instead of calling `MemoryExtensions.TryWrite`, you call
[Utf8.TryWrite](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/):

```csharp
public bool TryFormat(
    Span<byte> destination,
    out int bytesWritten,
    ReadOnlySpan<char> format,
    IFormatProvider provider) =>
        Utf8.TryWrite(destination, provider, $"({X}, {Y}, {Z})", out bytesWritten);
```

Same compiler magic, same ease of use. This time, the type of the interpolated
string handler is `Utf8.TryWriteInterpolatedStringHandler`. Its `AppendFormatted` method internally
calls `IUtf8SpanFormattable.TryFormat`, while the `AppendLiteral` method encodes its parameters to
UTF-8 at JIT time, which means you pay the runtime cost of transcoding UTF-16 to UTF-8 only once.

The only remaining problem is that even though you have `IUtf8SpanFormattable` as the
low-level building block, there is no string interpolation syntax that can use it internally to produce
the final formatted string. To be more specific, you can't say something like `$"Hello, {name}!"u8` and get
a byte array or a span as the result. There's a pretty easy workaround, though—simply call the `Utf8.TryWrite`
method one more time to write the interpolated string into a rented array:

```csharp
private static readonly Point3D _point = new(1.2, 2.5, 1.8);

[Benchmark]
public void Utf8TryWrite()
{
    var buffer = ArrayPool<byte>.Shared.Rent(128);
    Utf8.TryWrite(buffer, $"{_point}", out _);

    ArrayPool<byte>.Shared.Return(buffer);
}
```

If you benchmark this code, you will notice something fascinating.

| Method       | Mean     | Error   | StdDev  | Allocated |
|------------- |---------:|--------:|--------:|----------:|
| Utf8TryWrite | 127.5 ns | 1.28 ns | 1.19 ns |         - |

No allocations at all! And what's even more important, we didn't pay in implementation complexity—we used
the same string interpolation syntax we already know and love. Now let's see if we can find some real-world use
cases for UTF-8 string interpolation.

## A real-world example

To send a string over the network using `HttpClient`, you typically create an instance of `StringContent`.
The first thing the `StringContent` constructor does is call `Encoding.UTF8.GetBytes(content)`, which
allocates a new byte array. If you know the maximum possible size of your payload, you can avoid this
unnecessary allocation by renting a byte array, formatting the string as UTF-8 directly into it, and
then sending the data using `ByteArrayContent`:

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(MaxSize);
Utf8.TryWrite(buffer, $"{data}", out var bytesWritten);

using var client = new HttpClient();

var content = new ByteArrayContent(buffer, 0, bytesWritten);
var request = new HttpRequestMessage { Content = content };

await client.SendAsync(request);
ArrayPool<byte>.Shared.Return(buffer);
```

Let's measure the performance difference between creating `StringContent` and `ByteArrayContent`:

```csharp
private static readonly Line3D _line = new(
    new(1.23, 2.81, 3.56),
    new(0.85, 1.44, 4.32)
);

[Benchmark(Baseline = true)]
public StringContent StringContent()
{
    return new StringContent($"{_line}");
}

[Benchmark]
public void ByteArrayContent()
{
    var buffer = ArrayPool<byte>.Shared.Rent(1024);
    Utf8.TryWrite(buffer, $"{_line}", out int bytesWritten);

    _ = new ByteArrayContent(buffer, 0, bytesWritten);
    ArrayPool<byte>.Shared.Return(buffer);
}
```

| Method                | Mean     | Error   | StdDev  | Ratio | Allocated | Alloc Ratio |
|---------------------- |---------:|--------:|--------:|------:|----------:|------------:|
| StringContent         | 536.5 ns | 5.49 ns | 5.13 ns |  1.00 |     384 B |        1.00 |
| ByteArrayContent      | 428.1 ns | 2.47 ns | 2.19 ns |  0.80 |      64 B |        0.17 |

The difference is huge—UTF-8 string formatting allocates 83% less memory! And we didn't even
measure the best-case scenario, since our example used very short strings. The longer the
strings, the greater the improvements.

## The final implementation

Here is the full implementation of `Point3D` for reference.

```csharp
public record Point3D(double X, double Y, double Z)
    : ISpanFormattable, IUtf8SpanFormattable
{
    public override string ToString() => $"{this}";

    public string ToString(string format, IFormatProvider provider) => ToString();

    public bool TryFormat(
        Span<char> destination,
        out int charsWritten,
        ReadOnlySpan<char> format,
        IFormatProvider provider) =>
            destination.TryWrite(provider, $"({X}, {Y}, {Z})", out charsWritten);

    public bool TryFormat(
        Span<byte> destination,
        out int bytesWritten,
        ReadOnlySpan<char> format,
        IFormatProvider provider) =>
            Utf8.TryWrite(destination, provider, $"({X}, {Y}, {Z})", out bytesWritten);
}
```

## Conclusion

Modern .NET is really amazing. Whenever I think I don't need any further improvements in the
language or runtime, the .NET team surprises me with some new and useful high-performance
features. The fact that you can format strings with zero allocations using familiar syntax
is still mind-blowing to me! I can't wait to see how .NET 10 will delight us all.
