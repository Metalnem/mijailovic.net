---
layout: post
title: "Optimizing memory usage with modern .NET features"
image: /assets/img/2025-04-10-memory-optimizations-preview.png
date: 2025-04-10 10:00:00 +0200
---
After [my service migrated](https://devblogs.microsoft.com/dotnet/modernizing-push-notification-api-for-teams/)
from .NET Framework to .NET 8 (and later to .NET 9), it felt like a whole new world had opened to me. All
the modern .NET features that I had only been reading about on the [.NET Blog](https://devblogs.microsoft.com/dotnet/)
were finally available to me. Armed with Microsoft's continuous, fleet-wide performance profiler, I embarked on a journey
to find the places where my service was allocating the most memory and fix the unnecessary allocations by using the newly
available .NET APIs. In this blog post, I'll show you some common code patterns that can be found in .NET Framework code,
along with their modern, high-performance alternatives.

It's worth noting that before you start making improvements to your code,
you should make sure you are addressing a real performance problem.
Low-level optimizations can be incredibly addictive, and you could end up spending a lot of time on them without seeing
visible results. Every example I'm about to show you comes from a real-world memory allocation issue discovered using a
profiler, so you should conduct your own performance analysis as well ([PerfView](https://github.com/microsoft/perfview)
is a fantastic tool for that). Now let's dive in!

## String formatting

If your codebase has been around long enough, it likely contains many different ways of formatting strings:
concatenation using the plus operator, `StringBuilder` usages, and calls to `string.Join`, `string.Concat`,
and `string.Format` methods. Even though most of these are
[usually fine](https://learn.microsoft.com/en-us/dotnet/csharp/how-to/concatenate-multiple-strings), you can still
end up allocating much more memory than is really needed.

In almost all cases, [string interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation) should
be your preferred method for formatting strings. It's superior to other approaches in both speed and memory usage (if you want to
learn why, check out this [great post](https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/) written
by Stephen Toub). Not only that, but it also looks much nicer than building strings manually—especially when working with
interpolated multi-line strings. Thanks to the recently added
[raw string literals](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/raw-string),
it's never been easier to create complex templated strings like this one:

```csharp
string s = $"""
    Fancy report, ({DateTime.Now})
    Line 1: {Math.Pow(2, 16)}.
    Line 2: {new string('X', 5)}.
    Line 3: {RandomNumberGenerator.GetHexString(10)}.
    """;
```

## Collection capacity

Collections like `List<T>` and `Dictionary<TKey, TValue>` don't grow
magically—their implementations use a fixed-size array behind the scenes.
When that fixed-size array runs out of space for additional elements,
a new, larger array is allocated and existing elements are copied to it.
Most of the time, the compiler and the runtime handle this optimally, but
there is one surprising case where you need to assist them.

If you use a good ol' collection initializer, you might assume the compiler will statically determine the
initial collection capacity. It makes perfect sense, but it's also wrong. Take a look at this benchmark:
in the first case, we initialize the dictionary without specifying the capacity; in the second, we specify
the exact capacity we need:

```csharp
[Benchmark]
public Dictionary<string, string> DefaultCapacity()
{
    return new Dictionary<string, string>
    {
        ["1"] = "1",
        ["2"] = "2",
        ["3"] = "3",
        ["4"] = "4",
        ["5"] = "5",
        ["6"] = "6",
        ["7"] = "7",
        ["8"] = "8",
    };
}

[Benchmark]
public Dictionary<string, string> ExactCapacity()
{
    return new Dictionary<string, string>(8)
    {
        ["1"] = "1",
        ["2"] = "2",
        ["3"] = "3",
        ["4"] = "4",
        ["5"] = "5",
        ["6"] = "6",
        ["7"] = "7",
        ["8"] = "8",
    };
}
```

| Method          | Mean      | Error    | StdDev   | Gen0   | Allocated |
|---------------- |----------:|---------:|---------:|-------:|----------:|
| DefaultCapacity | 113.00 ns | 0.376 ns | 0.333 ns | 0.1185 |     992 B |
| ExactCapacity   |  65.57 ns | 0.623 ns | 0.521 ns | 0.0526 |     440 B |

Why is there such a huge difference in both CPU and memory usage? When you use a collection initializer,
the default constructor gets called, initializing the collection with zero capacity. Elements are then
added to the collection one by one, triggering the internal resizing algorithm as needed. Here's a simple
program to demonstrate how this works:

```csharp
var d = new Dictionary<int, int>();
Console.WriteLine($"Capacity: {d.Capacity,2}, Count: {d.Count}");

for (int i = 0; i < 8; ++i)
{
    d.Add(i, i);
    Console.WriteLine($"Capacity: {d.Capacity,2}, Count: {d.Count}");
}
```

This program prints the following results:

```
Capacity:  0, Count: 0
Capacity:  3, Count: 1
Capacity:  3, Count: 2
Capacity:  3, Count: 3
Capacity:  7, Count: 4
Capacity:  7, Count: 5
Capacity:  7, Count: 6
Capacity:  7, Count: 7
Capacity: 17, Count: 8
```

You can see that we unnecessarily allocated arrays of size 3 and 7, and that the final array is way
too large for a collection of 8 elements. As you already know from previous benchmark results, you
can avoid all this throwaway work by specifying the collection size in advance.

Is manually counting the number of elements in collection initializers really the best we can do?
For dictionaries, yes. For lists, there is a better way.
[Collection expressions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions)
are not only a cosmetic feature, but also the most performant way of initializing collections. Unlike
collection initializers, collection expressions set the exact capacity and are also much faster:

| Method                     | Mean     | Error    | StdDev   | Gen0   | Allocated |
|--------------------------- |---------:|---------:|---------:|-------:|----------:|
| InitializerDefaultCapacity | 57.46 ns | 0.244 ns | 0.228 ns | 0.0440 |     368 B |
| InitializerExactCapacity   | 26.59 ns | 0.109 ns | 0.102 ns | 0.0162 |     136 B |
| CollectionExpression       | 10.69 ns | 0.012 ns | 0.012 ns | 0.0163 |     136 B |

Of course, this doesn't mean that you should immediately update your entire codebase to use collection
expressions for all lists and set the initial capacity for all dictionaries.
What it means is that
if you use a lot of fixed-size collections in your hot code path and see a significant portion
of your allocations coming from `Resize` calls, there is an easy fix for that problem.

## Stack-allocated memory

There is one trick you won't be able to use often, but if the right conditions apply, it can lead to a useful performance
optimization. [Stackalloc](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc) allows
you to allocate a small block of memory on the stack and bypass garbage collection completely (the entire stack frame is
discarded after you exit the function). One example of its usage is computing cryptographic hashes. Their output size
is small and well-known, which means you can calculate them like this:

```csharp
Span<byte> hash = stackalloc byte[SHA256.HashSizeInBytes];
SHA256.HashData(data, hash);
```

Of course, this isn't the only use case—you can use `stackalloc` any time you need a small, temporary
buffer. But what does small even mean in this context? On Windows, it means less than 1MB, which is
the default stack size (though I'm not sure how up to date this information is). .NET itself uses a
[512-byte](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/String.Manipulation.cs)
limit internally:

```csharp
internal const int StackallocIntBufferSizeLimit = 128;
internal const int StackallocCharBufferSizeLimit = 256;
```

The examples from the
[documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/stackalloc)
set the limit somewhat higher, at 1,024 bytes, so both 512 bytes and 1,024 bytes should be perfectly safe.

## Case-insensitive hashing

Nowadays you probably know that you should avoid using `ToLower` or `ToUpper`
for case-insensitive string comparison, because these methods perform unnecessary
allocations. A better approach is to use the `StringComparison` overload, and an
[analyzer](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1862)
will remind you to do this. While this is all fairly obvious, what about case-insensitive hashing?
Even though the analyzer and the corresponding docs don't mention it, hash code calculation can
also be case-insensitive! If you are doing this:

```csharp
var hashCode = s.ToUpper().GetHashCode();
```

You can do this instead:

```csharp
var hashCode = s.GetHashCode(StringComparison.OrdinalIgnoreCase);
```

What about calculating the combined hash code of multiple objects? While you can't directly use
`HashCode.Combine` in this specific scenario, you can still create an instance of the `HashCode`
struct and add a string to it using the `StringComparer` overload. Pretty neat!

```csharp
HashCode hashCode = new();
hashCode.Add(s, StringComparer.OrdinalIgnoreCase);
```

## Hex conversion

For almost two decades, there was no good way to convert byte arrays to hexadecimal strings in .NET
(and no, [System.Runtime.Remoting.Metadata.W3cXsd2001.SoapHexBinary][SoapHexBinary] doesn't count).
Apart from rolling your own implementation (there are a million different ones all over the internet),
you had two available options: one bad and one terrible.

The bad one was `BitConverter.ToString`. For some reason, it was designed to generate a string
in which all hexadecimal pairs were separated by hyphens, so everyone had to use it like this:

```csharp
var upper = BitConverter.ToString(bytes).Replace("-", "");
var lower = BitConverter.ToString(bytes).Replace("-", "").ToLower();
```

If that's the bad option, what's the terrible one? Warning: graphic content ahead.

```csharp
string.Join("", bytes.Select(b => b.ToString("x2")));
```

Recent versions of .NET finally offer proper methods for hex conversion: `Convert.ToHex`
and `Convert.ToHexStringLower`. There is even an
[analyzer](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1872)
that will warn you if you are using the `BitConverter` pattern I described earlier, making it easy
to switch to the modern approach.

[SoapHexBinary]: https://learn.microsoft.com/en-us/dotnet/api/system.runtime.remoting.metadata.w3cxsd2001.soaphexbinary

## HttpContent JSON deserialization

You can easily shoot yourself in the foot when deserializing HTTP responses in
JSON format. For example, let's say you decide
to use `Json.NET`'s `JsonConvert.DeserializeObject`, the most well-known
method for parsing JSON data. Since that method only works with strings, you
need to read the HTTP response as a string, too. But if the HTTP response is
sufficiently large, your string will end up on the large object heap (yuck).

One way to avoid this problem is to use a complicated combination of
`StreamReader`, `JsonTextReader`, and `JsonSerializer` (documented
[here](https://www.newtonsoft.com/json/help/html/performance.htm#MemoryUsage)),
but a better solution is to simply use `System.Text.Json`. It's faster,
allocates less memory, and is easy to use. You can send the HTTP request,
receive the response, and deserialize the JSON content in one line of code.
That line also happens to be optimal in terms of performance!

```csharp
using var client = new HttpClient();
var result = await client.GetFromJsonAsync<Data>(url);
```

And if you throw in some compile-time
[source code generation](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation),
you get reflection-free deserialization code, ready for
[Native AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/) scenarios:

```csharp
[JsonSerializable(typeof(Data))]
public partial class DataContext : JsonSerializerContext { }

using var client = new HttpClient();
var result = await client.GetFromJsonAsync(url, DataContext.Default.Data);
```

Here are some performance numbers showing the difference between `Json.NET`, `System.Text.Json`,
and source-generated `System.Text.Json`. The benchmark is measuring the time to make an HTTP
request to a TCP socket listening on `localhost` and then parse the response.

| Method                         | Mean     | Error    | StdDev   | Gen0   | Allocated |
|------------------------------- |---------:|---------:|---------:|-------:|----------:|
| NewtonsoftJson                 | 78.85 us | 0.673 us | 0.629 us | 0.4883 |  12.02 KB |
| SystemTextJson                 | 69.06 us | 1.348 us | 1.324 us | 0.2441 |   6.26 KB |
| SourceGeneration               | 67.65 us | 1.085 us | 1.015 us | 0.2441 |   6.26 KB |

`System.Text.Json` is a winner, but the difference is not as dramatic as you might expect.
`Json.NET` is still a fine option—it's a well-optimized, mature library. If you are happy
with it, feel free to keep using it, just make sure you are using it correctly.

## Memory allocations in unexpected places

There are certain places where memory allocations definitely shouldn't be happening.
Here's an example from real-world code (the name of the class has been changed to protect
its real identity):

```csharp
public class MemoryEater
{
    public string Value => string.Concat(Base, ".", Extension);
    public bool Equals(MemoryEater other) => string.Equals(Value, other.Value);
}
```

You might notice a couple of issues here. The first one is that the property getter is allocating memory.
While I'm not aware of any official guidelines on avoiding allocations in getters, the callers usually
expect properties to behave as fields in disguise. That means it's common to see a property used in the
following way:

```csharp
if (!string.IsNullOrEmpty(instance.Value))
{
    DoSomething(instance.Value);
}
```

In our case, this will waste both CPU and memory, and if you don't know how the property getter is
implemented, you might not even realize there's potentially a hidden performance issue.

The second issue might be even worse: memory is being allocated in the `Equals` method. The number
of people who would be happy to get an `OutOfMemoryException` while comparing two objects
for equality is, you guessed it—exactly zero.

Fixing both problems is straightforward. Avoid complex properties by either pre-calculating
their values or converting them to methods (to signal to the callers that they are doing
some non-trivial amount of work). And definitely don't allocate memory in `Equals`. In the
case of the `MemoryEater` class, you can easily avoid the allocations by comparing the
individual components separately:

```csharp
public bool Equals(MemoryEater other)
{
    return Base == other.Base && Extension == other.Extension;
}
```

## Closing thoughts

It's been a blast being a .NET developer ever since .NET Core was released, especially
when you are into writing high-performance code. The .NET team's dedication to performance
is impressive, and .NET is getting faster and more fun to use with every new release. While
we are waiting for .NET 10, you can't go wrong with reading Stephen Toub's previous annual
[Performance Improvements in .NET](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/)
posts. Whether you want to learn more about performance optimization techniques, or just discover new
and incredibly fast .NET APIs, these posts are a gold mine of information, and I can't recommend them enough. 

<small><i>Endless thanks to my wife and editor Milica Miljkov, who somehow
always has the unwavering enthusiasm to sit with me for hours and edit
paragraphs destined to be read by maybe five people.</i></small>
