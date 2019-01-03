---
layout: post
title: "SharpFuzz: Bringing the power of afl-fuzz to .NET platform"
date: 2019-01-03 14:45:00 +0200
---
I'm a big believer in fuzzing (if you don't know what fuzzing is, now is
the best time to read my post [Going down the rabbit hole with go-fuzz],
even if you are not familiar with Go). Unfortunately, it never got popular
enough in the context of managed languages such as C# or Java. One of the
reasons is that fuzzers are usually security-oriented, and as such they are
often used with targets written in C/C++ to find memory-corruption
vulnerabilities, which is a class of problems that is completely eliminated
in managed programming languages.

The success of [go-fuzz] proved to me that coverage-guided fuzzing can be
surprisingly effective even outside the C/C++ world. Led by its example,
I've been thinking a lot over the last year about the possible approaches
to fuzzing .NET libraries. Today, I can finally present you [SharpFuzz].

[Going down the rabbit hole with go-fuzz]: {{ site.baseurl }}{% post_url 2017-07-29-go-fuzz %}
[go-fuzz]: https://github.com/dvyukov/go-fuzz
[SharpFuzz]: https://github.com/Metalnem/sharpfuzz

## Motivation

What types of problems could we possibly find by fuzzing .NET programs,
if we know that we don't have to worry about memory safety? My primary
goal was to look for bugs such as out-of-bounds array access, which
results in an **IndexOutOfRangeException**, or dereferencing a null
object reference, which results in a **NullReferenceException**.

C# also doesn't have [checked exceptions], which can sometimes be
problematic. If you are using some library method that can throw
an exception, you may want to catch it. But what do you catch? If
you are extremely lucky and the library is well-documented (which
is rarely the case), it might say that it only throws an
**IOException**, for example. However, the compiler doesn't
care about what documentation says, and the documentation can be
out of date. Finding this kind of mismatch between the documentation
and the actual behavior was the secondary goal of my research.

[checked exceptions]: https://www.artima.com/intv/handcuffsP.html

## History

[American fuzzy lop] is the most popular fuzzer today. It's using
compile-time instrumentation, which is why we can't directly apply
it to .NET programs, where just-in-time compiler is generating the
machine code at runtime. I knew that if all other methods failed,
I could always port AFL (or go-fuzz) to .NET, but that would be a
pretty time-consuming project. The goal of finding unexpected
exceptions was not *that* important to me, so I started searching
for an easier way to accomplish my task.

My initial idea was to try compiling .NET Core applications into
native executables using [CoreRT], an experimental ahead-of-time
compiler. One of the ways CoreRT can generate native code is by
using transpiler to convert IL to C++. That was exactly what I was
looking for, but there was a catch: [exception handling] is still
one of the big missing features in the C++ code generator.
Exceptions were *only the main topic* of my project, so I had to
abandon the CoreRT idea.

The next step was to find out if there exists a project that is
successfully using AFL to fuzz programs written in any managed
language, which is how I stumbled upon [Kelinci]. Kelinci is an
interface for running AFL on Java programs, and its approach to
fuzzing is something that I finally adopted for SharpFuzz (with
several important modifications).

[American fuzzy lop]: http://lcamtuf.coredump.cx/afl/
[CoreRT]: https://github.com/dotnet/corert/blob/master/Documentation/intro-to-corert.md
[exception handling]: https://github.com/dotnet/corert/issues/910
[Kelinci]: https://github.com/isstac/kelinci

## Technical details

Most of what I'll be talking about in this section can be
found in the [Technical whitepaper for afl-fuzz], which I
recommend you to read if you are interested in how AFL works.
If you are not interested in the technical details of SharpFuzz,
you can safely skip the rest of this section and go straight to
the usage details.

In short, AFL employs a "fork server" model, where the instrumented
binary forks itself into two processes during fuzzing. The server
process communicates with the afl-fuzz via anonymous pipes, while
the client process runs the target code and writes the captured
coverage results to shared memory. Given that afl-fuzz is using
only anonymous pipes and shared memory to communicate with the
fuzzed binary, we can easily simulate such behavior in C#.

The second part of the picture is the instrumentation. AFL
tracks the branch coverage in the target program by injecting
something similar to the following code at each branch point:

```c
cur_location = <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++; 
prev_location = cur_location >> 1;
```

This was also easily doable in C#. All I had to do was to rewrite
the IL of the target .NET assembly, injecting the equivalent IL
instructions where needed. Thanks to [Mono.Cecil], that part
was super simple (if you are interested in the exact details,
take a look at the [Method.cs] that does the instrumentation,
it's really short and easy to understand).

[Technical whitepaper for afl-fuzz]: http://lcamtuf.coredump.cx/afl/technical_details.txt
[Mono.Cecil]: https://github.com/jbevain/cecil
[Method.cs]: https://github.com/Metalnem/sharpfuzz/blob/master/src/SharpFuzz/Method.cs

## Usage

Detailed usage instructions for SharpFuzz can be found in
the [README] file of the GitHub repository (it's a very long
document, which is why I'm not including it in this post),
but I'm going to describe the general idea here anyway.

The first part of the fuzzing is to choose some .NET library
and instrument it using the [SharpFuzz.CommandLine] global
.NET tool. The second part is writing the fuzzing function.
Taking the [AngleSharp] HTML parsing library as an example,
you would create a new .NET console project, and write a
function looking something like this:

```csharp
using System.IO;
using AngleSharp.Parser.Html;
using SharpFuzz;

namespace AngleSharp.Fuzz
{
  public class Program
  {
    public static void Main(string[] args)
    {
      Fuzzer.Run(() =>
      {
        using (var file = File.OpenRead(args[0]))
        {
          new HtmlParser().Parse(file);
        }
      });
    }
  }
}
```

Any exception not caught in the function passed to **Fuzzer.Run**
will be reported to afl-fuzz as a crash. After instrumenting the
binary and writing the fuzzing function, you will also have to
create some test cases. They should be short and simple inputs
that are accepted as valid by your program. Here's
the HTML I'm using for fuzzing HTML parsing libraries:

```html
<html><body><h1>h1</h1><p>p</p></body></html>
```

After instrumenting the library, writing the fuzzing function,
and creating test cases, you will be ready to start the fuzzing.
The final step is to run afl-fuzz with the following command:

```shell
afl-fuzz -i testcases_dir -o findings_dir dotnet path_to_assembly @@
```

[README]: https://github.com/Metalnem/sharpfuzz/blob/master/README.md
[SharpFuzz.CommandLine]: https://www.nuget.org/packages/SharpFuzz.CommandLine/
[AngleSharp]: https://www.nuget.org/packages/AngleSharp/

## Results

Writing a fuzzing tool is all well and good, but it's not enough
without proving that it's effective in practice. I decided to test
SharpFuzz on some of the most popular NuGet libraries. To be included
in the initial testing batch, the library had to have more than 100K
users, and it also had to do some sort of complex input parsing.

My initial expectations were pretty low. I remember thinking that
I would call the project successful if I found maybe three of four
unexpected exceptions in total, but the actual results completely
blew me away:

- [AngleSharp: HtmlParser.Parse throws InvalidOperationException](https://github.com/AngleSharp/AngleSharp/issues/735)
- [ExcelDataReader: ExcelReaderFactory.CreateBinaryReader can throw unexpected exceptions](https://github.com/ExcelDataReader/ExcelDataReader/issues/383)
- [ExcelDataReader: ExcelReaderFactory.CreateBinaryReader throws OutOfMemoryException](https://github.com/ExcelDataReader/ExcelDataReader/issues/382)
- [ExCSS: StylesheetParser.Parse throws ArgumentOutOfRangeException](https://github.com/TylerBrinks/ExCSS/issues/101)
- [Google.Protobuf: MessageParser.ParseFrom throws unexpected exceptions (C#)](https://github.com/protocolbuffers/protobuf/issues/5513)
- [GraphQL-Parser: Parser.Parse takes around 18s to parse the 58K file](https://github.com/graphql-dotnet/parser/issues/22)
- [GraphQL-Parser: Parser.Parse throws ArgumentOutOfRangeException](https://github.com/graphql-dotnet/parser/issues/21)
- [Jil: JSON.DeserializeDynamic throws ArgumentException](https://github.com/kevin-montrose/Jil/issues/316)
- [Jint: Engine.Execute can throw many unexpected exceptions](https://github.com/sebastienros/jint/issues/571)
- [Jint: Engine.Execute terminates the process by throwing a StackOverflowException](https://github.com/sebastienros/jint/issues/572)
- [Json.NET: JsonConvert.DeserializeObject can throw several unexpected exceptions](https://github.com/JamesNK/Newtonsoft.Json/issues/1947)
- [Jurassic: ScriptEngine.ExecuteFile hangs permanently instead of throwing JavaScriptException](https://github.com/paulbartrum/jurassic/issues/138)
- [Jurassic: ScriptEngine.ExecuteFile throws FormatException](https://github.com/paulbartrum/jurassic/issues/137)
- [LumenWorks CSV Reader: CsvReader.ReadNextRecord throws IndexOutOfRangeException](https://github.com/phatcher/CsvReader/issues/67)
- [Markdig: Markdown.ToHtml hangs permanently](https://github.com/lunet-io/markdig/issues/278)
- [Markdig: Markdown.ToHtml throws ArgumentOutOfRangeException](https://github.com/lunet-io/markdig/issues/275)
- [Markdig: Markdown.ToHtml throws IndexOutOfRangeException](https://github.com/lunet-io/markdig/issues/276)
- [Markdig: Markdown.ToHtml throws NullReferenceException](https://github.com/lunet-io/markdig/issues/277)
- [MarkdownSharp: Markdown.Transform hangs permanently](https://github.com/StackExchange/MarkdownSharp/issues/8)
- [MessagePack for C#: MessagePackSerializer.Deserialize hangs permanently](https://github.com/neuecc/MessagePack-CSharp/issues/359)
- [MessagePack for CLI: Unpacking.UnpackObject hangs permanently](https://github.com/msgpack/msgpack-cli/issues/309)
- [MessagePack for CLI: Unpacking.UnpackObject throws several unexpected exceptions](https://github.com/msgpack/msgpack-cli/issues/311)
- [Mono.Cecil: ModuleDefinition.ReadModule can throw many (possibly) unexpected exceptions](https://github.com/jbevain/cecil/issues/556)
- [Mono.Cecil: ModuleDefinition.ReadModule hangs permanently](https://github.com/jbevain/cecil/issues/555)
- [NCrontab: CrontabSchedule.Parse throws OverflowException instead of CrontabException](https://github.com/atifaziz/NCrontab/issues/43)
- [NUglify: Uglify.Js hangs permanently](https://github.com/xoofx/NUglify/issues/63)
- [Open XML SDK: Add some security/fuzz testing](https://github.com/OfficeDev/Open-XML-SDK/issues/441)
- [protobuf-net: Serializer.Deserialize can throw many unexpected exceptions](https://github.com/mgravell/protobuf-net/issues/481)
- [protobuf-net: Serializer.Deserialize hangs permanently](https://github.com/mgravell/protobuf-net/issues/479)
- [SharpCompress: Enumerating ZipArchive.Entries collection throws NullReferenceException](https://github.com/adamhathcock/sharpcompress/issues/431)
- [SharpZipLib: ZipInputStream.GetNextEntry hangs permanently](https://github.com/icsharpcode/SharpZipLib/issues/300)
- [SixLabors.Fonts: FontDescription.LoadDescription throws ArgumentException](https://github.com/SixLabors/Fonts/issues/96)
- [SixLabors.Fonts: FontDescription.LoadDescription throws NullReferenceException](https://github.com/SixLabors/Fonts/issues/97)
- [SixLabors.ImageSharp: Image.Load terminates the process with AccessViolationException](https://github.com/SixLabors/ImageSharp/issues/798)
- [SixLabors.ImageSharp: Image.Load throws NullReferenceException](https://github.com/SixLabors/ImageSharp/issues/797)
- [Utf8Json: JsonSerializer.Deserialize can throw many unexpected exceptions](https://github.com/neuecc/Utf8Json/issues/142)
- [Web Markup Minifier: HtmlMinifier.Minify hangs permanently](https://github.com/Taritsyn/WebMarkupMin/issues/73)
- [YamlDotNet: YamlStream.Load terminates the process with StackOverflowException](https://github.com/aaubry/YamlDotNet/issues/375)
- [YamlDotNet: YamlStream.Load throws ArgumentException](https://github.com/aaubry/YamlDotNet/issues/374)

All of the 25 tested libraries had at least one issue!
But not all issues are created equal, so let's dive
deeper into the results.

### Unexpected exceptions

Unexpected exceptions were what I was looking for in the first
place, and I found a lot of them. In most cases it was some sort
of **ArgumentException**, **IndexOutOfRangeException**, or
**NullReferenceException**. Most of the libraries have their
own custom exception type, and this proves that it's very
difficult to rely on them.

### Hangs

I also found a lot of hangs, both temporary and permanent,
even though I didn't expect them at all. They are much more
severe than unexpected exceptions, which you can at least
catch. There is nothing simple you can do to combat hangs
as a library user.

### Uncatchable exceptions

These are the most rare, but also the most devastating.
Even if you decide to fight unexpected exceptions by
catching all exceptions, **AccessViolationException**
and **StackOverflowException** will still kill your
process mercilessly.

## Conclusion

One of the problems with software testing is that developers
can't possibly create test cases that are as malicious as
something that fuzzer can come up with. No matter how many
unit tests you write, you would probably never test your
JavaScript parsing code with any of the following snippets
(each one of these broke a different .NET JavaScript engine):

```
~ (WE0=1)--- l('1');
for(a=0;a<2;a++[1])1
switch=1) eval('1');
```

I hope that I've convinced you with this that the fuzzing
is not just necessary, but that it's also incredibly fun.
Getting started with fuzzing can still be difficult, so
if you are running some large or important project, and are
interested in fuzzing, I'm ready to assist you in setting
up SharpFuzz.

This is just the beginning of the story of .NET fuzzing.
I'll be testing even more NuGet libraries in the future,
but I will also fuzz the .NET standard library, which
will be the topic of one of my future posts. Happy fuzzing!
