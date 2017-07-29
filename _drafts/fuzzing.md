---
layout: post
title: "Adventures in go-fuzz"
---
[Fuzzing](https://en.wikipedia.org/wiki/Fuzzing)
has been a well known automated testing technique for decades, especially
among security professionals. It used to be "dumb", meaning it was only
capable of generating random data and feeding it to the program that
was being tested. It has evolved a lot in the last few years. Modern
fuzzing tools now track code coverage and leverage symbolic execution
to systematically explore different code paths in the program (watch
[The Smart Fuzzer Revolution](https://www.youtube.com/watch?v=g1E2Ce5cBhI)
if you are interested in the state of the art in the field).
[american fuzzy lop](http://lcamtuf.coredump.cx/afl/) brought fuzzing
to the masses. It didn't have any revolutionary feature, but thanks
to it's ease of use, we are not living in the fuzzing renaissance.

Inspired by AFL, Dmitry Vyukov created
[go-fuzz](https://github.com/dvyukov/go-fuzz), which is a fuzz testing
tool for the Go programming language. It's one of the most useful tools
in the Go toolchain, but it's not as widely known as it should be.
I noticed it a long time ago and immediately knew that it's pretty
impressive library, but I didn't have the time to play with it back then.
Ever since that, it’s GitHub page has been sitting in my bookmarks, until
few weeks ago I found [this great article](https://medium.com/@dgryski/go-fuzz-github-com-arolek-ase-3c74d5a3150c)
that explained go-fuzz usage step by step. Now I had no excuse for not
being familiar with go-fuzz, so I decided to finally try it.

## Getting started

Other people have already written good go-fuzz tutorials, so I won't go
into all details of using go-fuzz in this post, but I will point you
to the articles and talks that will get you up to speed quickly.

The best way to get started with go-fuzz is to read Damian Gryski's
great article [go-fuzz github.com/arolek/ase](https://medium.com/@dgryski/go-fuzz-github-com-arolek-ase-3c74d5a3150c).
Another great starting point is [GitHub page](https://github.com/dvyukov/go-fuzz) for go-fuzz.
For a slightly more advanced usage, you should read Filippo Valsorda's
adventures with DNS parsing in his blog post
[DNS parser, meet Go fuzzer](https://blog.cloudflare.com/dns-parser-meet-go-fuzzer/)
(if you prefer watching videos, you can find his GopherCon 2015 talk
about it [here](https://www.youtube.com/watch?v=kOZbFSM7PuI)). Finally,
if you want to know more about go-fuzz internals, watch the author's
presentation about [Go Dynamic Tools](https://www.youtube.com/watch?v=a9xrxRsIbSU).

## Fuzzing Runtastic Archiver

Naturally, I wanted to start by fuzzing my own Go programs.
Fuzzing is the most effective for finding bugs in code that parses complex
data (it can be used in a lot of other ways, but that won’t be the topic
today). The closest thing to that was parsing custom binary protocol in
[Runtastic Archiver](https://github.com/Metalnem/runtastic), which is
a tool that downloads [Runtastic](https://www.runtastic.com/) activities
and converts them to TCX or GPX format. Activities are retrieved from
the server in JSON format, but the GPS trace field is actually a binary
blob that had to be parsed according to some serialization rules.
The function that parses the trace has the following signature:

```go
func parseGPSData(trace string) ([]gpsPoint, error)
```

Functions like this are the ideal candidates for fuzzing in most
situations. Unfortunately, the binary format in this case was super
simple, and the function was really short (just few dozen lines
of code), so I didn't expect to find any crashes in it. Because
I didn't have a better candidate, I decided to fuzz it anyway.
The great thing about go-fuzz is that you can place the fuzz function
anywhere in your code. That means you can easily test your private
functions (notice that *parseGPSData* is private), thus focusing
on the areas that are most likely to produce bugs. Here is the
fuzz function that I came up with:

```go
func Fuzz(data []byte) int {
  if _, err := parseGPSData(string(data)); err != nil {
    return 0
  }

  return 1
}
```

Fuzzing functions follow the same general pattern in most cases.
You call the function that you want to test, return 0 if the input was
invalid, and 1 if it was valid. go-fuzz will focus on valid inputs,
ignore invalid ones, and hopefully catch some inputs that cause panic
during the process.

Here is another similar example. Few months ago, I was playing with
various [parsing algorithms](https://github.com/Metalnem/parsing-algorithms)
(precedence climbing and Shunting yard), so I wanted to test that package,
too. Fuzz function was following the same pattern:

```go
func Fuzz(data []byte) int {
  if _, err := New().Parse(string(data)); err != nil {
    return 0
  }

  return 1
}
```

In both cases, go-fuzz didn't find anything. Not because of my mad
programming skills, mind you, but because the code that did the parsing was
in both cases really simple, so there was never a good chance that it
could crash. This was not a satisfactory conclusion; I needed to find
some other libraries to fuzz.

## Fuzzing UniDoc

I started looking through the list of all dependencies in my Go
projects on GitHub, and the ideal candidate for fuzzing showed up immediately.
It was [UniDoc](https://github.com/unidoc/unidoc), the PDF library for Go
(it's also the engine behind [FoxyUtils](https://foxyutils.com/)).
I was using it for decrypting and merging the individual magazine pages
in both [Future plc downloader](https://github.com/Metalnem/future-plc-downloader)
and [Zinio DRM removal](https://github.com/Metalnem/zinio).
UniDoc is a really great library, but PDF is also a incredibly complex
format, so I was almost 100% certain that if the library’s authors haven’t
fuzzed it, go-fuzz would find some bugs in it. Again, writing the Fuzz
function was pretty easy—I just had to convert the input bytes into *io.ReadSeeker*
and parse it by calling *pdf.NewPdfReader*.

```go
func Fuzz(data []byte) int {
  b := bytes.NewReader(data)

  if _, err := pdf.NewPdfReader(b); err != nil {
    return 0
  }

  return 1
}
```

The other important thing was to find the good initial corpus of PDF
files. I just did a Google search for “PDF sample” and downloaded
the first three results. Everything was ready for running go-fuzz. After only
a few minutes, first crashers started to appear. I left go-fuzz running for
about an hour, after which the final count of crashing inputs was well
over 50. Most of them were actually duplicates. go-fuzz keeps only crashes
with unique stack traces, but the unique stack traces don’t guarantee that
the crash wasn’t happening in the same place. In the case of UniDoc,
most of the crashes were in the same function, but the call could happen
at arbitrary depth in the call stack, because of the hierarchical nature of
PDF format. After looking through all the crashers, number of unique bugs
settled at four. Here are all of them:

[panic: interface conversion: pdf.PdfObject is nil, not *pdf.PdfObjectInteger](https://github.com/unidoc/unidoc/issues/77)  
[panic: runtime error: makeslice: len out of range](https://github.com/unidoc/unidoc/issues/78)  
[panic: runtime error: invalid memory address or nil pointer dereference](https://github.com/unidoc/unidoc/issues/79)  
[runtime: goroutine stack exceeds 1000000000-byte limit](https://github.com/unidoc/unidoc/issues/80)  

Within less than two days, library authors fixed all of them, but that
was not all: I suggested that they should consider fuzzing the library
with larger initial corpus, which helped them find
[six additional bugs](https://github.com/unidoc/unidoc/issues/81)!
Thanks to go-fuzz, we now have much more reliable PDF library for Go.

At this moment I was wondering whether I should continue play with
go-fuzz and some different libraries, or just call it a day.
It was a fun and useful process, so I decided to test one more library.

## Fuzzing goftp

The next interesting library was [goftp](https://github.com/jlaffaye/ftp),
which I used to write [LFTP server](https://github.com/Metalnem/lftp-server).
Writing FTP clients involves some non-trivial parsing, so even though the
library was small, I was hoping I would find something interesting. This time,
the process was a little bit more involved. The library is designed to connect
to real FTP servers, but I wanted to test only the parsing code, which was
internal to the package. As I said earlier, the great thing about go-fuzz
is that you can easily test even private methods. Combine that with
the open source nature of Go packages and you’ll get the ability to `go get`
any package and test its private methods, which was exactly what I did.
In the parse.go file I found this function:

```go
func parseListLine(line string) (*Entry, error)
```

That's exactly what I was looking for! Now I had to find some test inputs.
The most common way to find them is to just take the data that you feed
your tests with and create a separate file for each test input. Fortunately,
the FTP package was well tested, so I had more that 20 tests in the initial
corpus. Fuzz function was again pretty easy to write:

```go
func Fuzz(data []byte) int {
  if _, err := parseListLine(string(data)); err != nil {
    return 0
  }

  return 1
}
```

After few minutes, go-fuzz found two crashing inputs, but stayed
at that number even after one hour. Finding two crashes was
good enough for me, so I stopped the fuzzing process and started
looking through inputs that caused them. In the first file, the
input that broke the line parser was `000000000x ` (notice the
whitespace at the end—this would be really difficult to find without
fuzzing). I filed an [issue](https://github.com/jlaffaye/ftp/issues/97)
on GitHub, and it was quickly resolved. But the second crash turned
out to be much more interesting.

## Crash in the standard library

Here is the input that caused the second crash:

```
-000000000 0 r 0 0 --- 4 0000
```

And here’s the stack trace after the crash:

```
panic: runtime error: index out of range

goroutine 1 [running]:
time.parse(0x10e4643, 0x13, 0xc42000c320, 0x12, 0x115d800, 0x115fa80, 0xc42000c320, 0xc42000c328, 0xc420049cd0, 0x0, ...)
	/usr/local/Cellar/go/1.8.3/libexec/src/time/format.go:1015 +0x2c77
time.Parse(0x10e4643, 0x13, 0xc42000c320, 0x12, 0xc42000c320, 0x12, 0xc0, 0xc0, 0x10bd700)
	/usr/local/Cellar/go/1.8.3/libexec/src/time/format.go:743 +0x68
github.com/jlaffaye/ftp.(*Entry).setTime(0xc4200163c0, 0xc42008a050, 0x3, 0x7, 0x0, 0x0)
	/Users/Metalnem/Go/src/github.com/jlaffaye/ftp/parse.go:237 +0x1c9
github.com/jlaffaye/ftp.parseLsListLine(0x10e64b4, 0x1d, 0x114c200, 0xc4200107d0, 0xc420010701)
	/Users/Metalnem/Go/src/github.com/jlaffaye/ftp/parse.go:135 +0x575
github.com/jlaffaye/ftp.parseListLine(0x10e64b4, 0x1d, 0x100499c, 0xc42001a0b8, 0x0)
	/Users/Metalnem/Go/src/github.com/jlaffaye/ftp/parse.go:209 +0x68
github.com/jlaffaye/ftp.ParseListLine(0x10e64b4, 0x1d, 0x10c63e0, 0xc420014660, 0x0)
	/Users/Metalnem/Go/src/github.com/jlaffaye/ftp/parse.go:218 +0x35
main.main()
	/Users/Metalnem/Go/src/github.com/jlaffaye/ftp/main/main.go:11 +0x3a
exit status 2
```

Wait, what? Top of the stack was not in the FTP library, but in
src/time/format.go file, which belongs to the Go standard library!
That was the last thing I expected—the Go standard library has been
extremely well tested with go-fuzz.

I called the *parseListLine* function again with the line that was
causing the crash, but this time with the debugger attached,
because I wanted to isolate the string that made the *time.Parse*
function panic. Here is the call that was causing the panic:

```go
time.Parse("_2 Jan 06 15:04 MST", "4 --- 00 00:00 GMT")
```

Would you ever be able to come up with such silly test case
on your own? Another victory for go-fuzz! Again, I filed an
[issue](https://github.com/golang/go/issues/21113) on GitHub.
It was marked as a release-blocker, which meant I was affecting
millions of lives by delaying the next release of Go for at
least 10 minutes!

## Conclusion

>If you didn't fuzz it, you can't say it's correct.
>
> — <cite>Dmitry Vyukov</cite>

It's really difficult to catch all bugs without fuzzing, no matter how hard
you try to test your software. Thanks to go-fuzz, fuzz testing has never been
easier ([proposals](https://docs.google.com/document/u/1/d/1zXR-TFL3BfnceEAWytV8bnzB2Tfp6EPFinWVJ5V4QC8/pub)
exist to make it even easier by adding fuzzing support as first-class citizen
to Go tooling). You should consider fuzzing not just your own code, but also
your dependencies—you can always find some interesting bugs that way, as you
have seen in this post. Also, don't forget to add your findings to go-fuzz
[list of trophies](https://github.com/dvyukov/go-fuzz#trophies). Happy fuzzing!
