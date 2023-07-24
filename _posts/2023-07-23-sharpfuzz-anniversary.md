---
layout: post
title: "Five years of fuzzing .NET with SharpFuzz"
date: 2023-07-23 19:00:00 +0200
---
It's been almost five years since I created
[SharpFuzz](https://github.com/Metalnem/sharpfuzz), the only .NET
coverage-guided fuzzer. I already have a blog post on how it
works, what it can do for you, and what bugs it found, so
check it out if this is the first time you hear about SharpFuzz:

[SharpFuzz: Bringing the power of afl-fuzz to .NET platform]({{ site.baseurl }}{% post_url 2019-01-03-sharpfuzz %})

A lot of interesting things have happened since then. SharpFuzz now
works with libFuzzer, Windows, and .NET Framework. And it can finally
fuzz the .NET Core base-class library! The whole fuzzing process has
been dramatically simplified, too.

Not many people are aware of all these developments, so I decided
to write this anniversary blog post and showcase everything SharpFuzz
is currently capable of.

## Trophies

The list of bugs found by SharpFuzz has been growing steadily and it now
contains more than 80 entries. I'm pretty confident that some of the bugs
in the .NET Core standard library would have been impossible to discover
using any other testing method:

- [BigInteger.TryParse out-of-bounds access](https://github.com/dotnet/runtime/issues/28652)
- [Double.Parse throws AccessViolationException on .NET Core 3.0](https://github.com/dotnet/runtime/issues/28872)
- [G17 format specifier doesn't always round-trip double values](https://github.com/dotnet/runtime/issues/28703)

As you can see, SharpFuzz is capable of finding not only crashes, but
also correctness bugs—the more creative you are in writing your fuzzing
functions, the higher your chances are for finding an interesting bug.

SharpFuzz can also find serious security vulnerabilities.
I now have two CVEs in my trophy collection:

- [CVE-2019-0980: .NET Framework and .NET Core Denial of Service Vulnerability](https://msrc.microsoft.com/update-guide/en-us/vulnerability/CVE-2019-0980)
- [CVE-2019-0981: .NET Framework and .NET Core Denial of Service Vulnerability](https://msrc.microsoft.com/update-guide/en-us/vulnerability/CVE-2019-0981)

If you were ever wondering if fuzzing managed languages makes sense,
I think you've got your answer right here.

## Easier fuzzing

Initial SharpFuzz usage instructions were
[unnecessarily complicated](https://github.com/Metalnem/sharpfuzz/blob/master/docs/legacy-usage-instructions.md)
and full of error-prone, manual steps. I decided to completely rewrite these
guidelines and also write a PowerShell script that correctly configures
all parameters behind the scenes, making the whole fuzzing process way simpler.
You no longer have to choose which assembly to instrument, which memory limit
to use, or how to configure the execution timeout. You only need to:

1. Create the fuzzing project and write your fuzzing function.
2. Create one or more test cases.
3. Run the [fuzz.ps1](https://github.com/Metalnem/sharpfuzz/raw/master/scripts/fuzz.ps1) script like this:

```shell
pwsh scripts/fuzz.ps1 YourFuzzingProject.csproj -i Testcases
```

That's all! For more details, check out the brand-new
[usage](https://github.com/Metalnem/sharpfuzz/blob/master/README.md#usage) section
in the [README](https://github.com/Metalnem/sharpfuzz/blob/master/README.md) file.
The new fuzzing script has made fuzzing much easier for me, and I hope everyone
else will benefit from it, too.

## libFuzzer

In addition to AFL, SharpFuzz now supports [libFuzzer](https://llvm.org/docs/LibFuzzer.html)
as a fuzzing engine. If you are interested in technical implementation details, check out the
[libfuzzer-dotnet](https://github.com/Metalnem/libfuzzer-dotnet) repository. If you just want
to start using libFuzzer, usage instructions are available
[here](https://github.com/Metalnem/sharpfuzz/blob/master/docs/libFuzzer.md). AFL and libFuzzer
have similar capabilities, so on its own, using libFuzzer instead of AFL doesn't really
bring you any benefits (it might be slightly easier to use, though, because you only have to
download a single binary and you are ready to go). However, libFuzzer support in SharpFuzz has
unlocked some exciting new possibilities: native Windows support and fuzzing .NET framework
libraries.

## Windows

After I finished writing the libFuzzer driver, I had completely forgotten about it.
I joined Microsoft later that year, and a few months into my Microsoft career, I stumbled
upon an internal fuzzing channel, where I discovered that there was a team in Microsoft
that was working on porting my SharpFuzz libFuzzer driver to Windows! They not only
wrote the port, but they were also happy to open source it. Huge thanks to Joe
Ranweiler and the MORSE team for what they have done—I'm deeply thankful for their
contribution, and I can't emphasize enough how important it was for the SharpFuzz
users. Native Windows support has made fuzzing much more accessible: despite .NET
Core's cross-platform success, most .NET users are probably still on Windows.

## .NET Framework

Everyone is moving to .NET Core these days. But the migration can be slow, and
there are many libraries that might never be ported to .NET Core. With
libFuzzer and Windows support, SharpFuzz can now fuzz .NET Framework libraries,
too! I didn't have to do anything special to enable this feature, though:
SharpFuzz has been targeting .NET Standard since the beginning. The only thing
needed to support .NET Framework was the fuzzing engine that could run on Windows,
and as you know from the previous section, libFuzzer satisfies that requirement.

## Fuzzing .NET Core standard library

My initial attempts to fuzz the .NET Core standard library (aka base-class library or BCL) failed,
because I thought that modifying mixed-mode assemblies was impossible (and all official .NET assemblies
are built as mixed-mode assemblies). But I really wanted to find bugs in the .NET standard library, so
I ultimately figured out how to fuzz it by doing some godawful hacks like downloading the IL-only,
nightly packages from a special NuGet feed used by the .NET team, and building the .NET runtime repo
in managed-only mode (my eyes are now bleeding from trying to read the
[usage instructions](https://github.com/Metalnem/sharpfuzz/blob/master/docs/fuzzing-dotnet-core.md)
I wrote about this procedure back in the day). Unfortunately, this solution was short-lived,
because the .NET team stopped publishing the IL-only packages. They also changed the way
.NET runtime was built, and I couldn't figure out how to build it in managed-only mode
again. This marked the end of my .NET Core BCL fuzzing attempts.

...until a few weeks ago. I was sitting in a coffee shop with a renewed
enthusiasm for building the .NET runtime when it suddenly dawned on me: why do
I even think that instrumenting mixed-mode assemblies is impossible? What if I
[unlearned](https://sive.rs/unlearning) that belief and started from scratch?
A few minutes later, I realized that the impossible was not only
[possible](https://github.com/0xd4d/dnlib/issues/305) (at least on Windows),
but it took me only a few moments to implement. I still don't really know how
mixed-mode assemblies work, but I do know that I can easily strip the native code
from them, and nothing will break. It's funny how sometimes you need to wait for
years for things to fall into place.

If you want to get started with fuzzing the .NET BCL, you can find plenty of
examples in my [dotnet-fuzzers](https://github.com/Metalnem/dotnet-fuzzers)
repo (contributions are welcome). Everything you need to get started is there:
fuzzing projects, dictionaries, and PowerShell commands. For example, if you want to fuzz
the [Uri.TryCreate](https://learn.microsoft.com/en-us/dotnet/api/system.uri.trycreate)
method, you simply need to clone the repo and run the following script:

```powershell
.\fuzz.ps1 `
  -project .\src\UriFuzzer\UriFuzzer.csproj `
  -corpus .\src\UriFuzzer\Testcases\ `
  -targetDlls System.Private.Uri.dll
```

Unfortunately for us bug finders, the .NET team has been doing a really great job in
recent years, so discovering new bugs in .NET Core has become more difficult. Every time
I thought "there must be a bug here", I was disappointed because the code turned out to be
[remarkably well designed](https://github.com/dotnet/designs/blob/main/accepted/2020/asnreader/asnreader.md).
Well done, .NET team, I hope you are happy for basically ruining my life.

## Community support

It makes me really happy to see so many awesome people participating in the SharpFuzz
development. I'm profoundly grateful to [Joe Ranweiler](https://github.com/ranweiler) and the
[MORSE](https://news.microsoft.com/source/features/innovation/morse-microsoft-offensive-research-security-engineering/)
team, [Günther Foidl](https://github.com/gfoidl), [Ulrich Fourier](https://github.com/ufo95),
[George Pollard](https://github.com/Porges), and [p4fg](https://github.com/p4fg) for their contributions.
Some people have written [blog posts](http://writeasync.net/?p=5714) about SharpFuzz,
and it was even a topic of a [research project](https://eprints.ost.ch/id/eprint/934/1/HS%202020%202021-SA-EP-PREMANANTHAN-SUNDRALINGAM-Moro-Visual%20Computing%20%20AR%20%20%20App.pdf)
(it's in German, though). Thank you all for being such a great community!

## Conclusion

After taking a break from fuzzing for several years, I'm again actively
working on SharpFuzz. There is still plenty of important and fun work in
.NET fuzzing (for example, ASP.NET Core fuzzing and
[structure-aware fuzzing](https://github.com/google/fuzzing/blob/master/docs/structure-aware-fuzzing.md)),
so expect more posts in the future. In the meantime, enjoy fuzzing!

<small><i>Big thanks to my loyal sidekick Milica Miljkov for her continuous support in my blogging efforts.</i></small>
