---
layout: post
title: "Error handling patterns in Go"
---
One of the main strengths of the Go programming language is its error model.
It's not my favorite—that would be something like
[this](http://joeduffyblog.com/2016/02/07/the-error-model/)-but it's still
one of the best on the market. Much has been written previously about best
practices for error handling in Go
([Error handling and Go](https://blog.golang.org/error-handling-and-go) and
[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)
are both good resources). In this post I will talk about some error handling
patterns that are less known, but nevertheless important for writing reliable
and simple Go code.

## Always check for errors

Almost everybody agrees that you should always check for errors in Go
(tools like [errcheck](https://github.com/kisielk/errcheck) can help
you with that). Same holds for releasing resources using defer
statement. Situation becomes more interesting when you combine these
two—what are we supposed to do with errors returned from functions
called inside defer statement? One pattern you see often is this:

```go
resp, err := http.Get(url)

if err != nil {
	return err
}

defer resp.Body.Close()
```

Even though the Close call could return an error, we can ignore it
without sacrificing correctness. That doesn't mean we can always
ignore errors in deferred calls. Consider this function for writing
some data to a file (let's ignore for the moment that ioutil.WriteFile
function does almost the same thing):

```go
func WriteFile(filename string, data []byte) error {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer f.Close()

  _, err = f.Write(data)
  return err
}
```

This looks fine on the surface, but it actually has a serious
flaw. Linux man page for close system call says this:

>Not checking the return value of close() is a common but nevertheless
serious programming error. It is quite possible that errors on a previous
write(2) operation are first reported at the final close(). Not checking
the return value when closing the file may lead to silent loss of data.

Now that we know we have to check for errors even when calling Close,
what's the best way to do that? Here is one of the possible correct
solutions:

```go
func WriteFile(filename string, data []byte) (err error) {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
      err = cerr
    }
  }()

  _, err = f.Write(data)
  return err
}
```

In this version, we are still calling Close in the defer statement
(it was not strictly necessary to use the defer statement here because
we only have one Write after if—the code would have been simpler if
we had just reordered them—but it's much safer approach in general),
but this time we are correctly checking for errors and updating
function's return value if there was an error during the Close call.
To do that, we had to modify function's signature to use a named
return (see this great [post](https://blog.golang.org/defer-panic-and-recover)
if you are not familiar with it).

This approach is correct, but the code has become much uglier: we now
have five lines of code just for handling situation that is not the primary
task of our function, and we also have to keep track of two different
errors instead of just one. If you have multiple resources you have to
release in this way, it gets even uglier. I usually solve this problem
by defining helper function called safeClose:

```go
func safeClose(c io.Closer, err *error) {
  if cerr := c.Close(); cerr != nil && *err == nil {
    *err = cerr
  }
}

func WriteFile(filename string, data []byte) (err error) {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer safeClose(f, &err)

  _, err = f.Write(data)
  return err
}
```

The code is almost exactly the same as in the first example, but
this time it's actually correct. I was thinking about making a
public package that exports this function, but decided against
it because the function is not good enough abstraction to be
made public.

If you are not convinced that all of this is necessary, take a look
at the implementation of ioutil.WriteFile from the standard library:

```go
func WriteFile(filename string, data []byte, perm os.FileMode) error {
  f, err := os.OpenFile(filename, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, perm)
  if err != nil {
    return err
  }
  n, err := f.Write(data)
  if err == nil && n < len(data) {
    err = io.ErrShortWrite
  }
  if err1 := f.Close(); err == nil {
    err = err1
  }
  return err
}
```

It doesn't use defer, but the approach is the same. Even if there
were no write errors, that doesn't mean your writes were successful:
you still have to check the return value of the Close call.

## Don't blindly use err != nil

Hopefully, we agree that you should always check for errors. That
doesn't mean you have to place `if err != nil { ... }` checks everywhere:
they can obscure the meaning of your code and make real bugs harder to
spot. Consider this (abridged) example from the real-world code that
I wrote (original structure had even more fields):

```go
type point struct {
  Longitude     float32
  Latitude      float32
  Distance      int32
  ElevationGain int16
  ElevationLoss int16
}
```

Here we have a structure with bunch of fields with different types.
They are encoded in Big Endian format on the server and sent to a client. To
decode single point from the input stream, you might initially write code
looking something like this:

```go
func parse(r io.Reader) (*point, error) {
  var p point

  if err := binary.Read(r, binary.BigEndian, &p.Longitude); err != nil {
    return nil, err
  }

  if err := binary.Read(r, binary.BigEndian, &p.Latitude); err != nil {
    return nil, err
  }

  if err := binary.Read(r, binary.BigEndian, &p.Distance); err != nil {
    return nil, err
  }

  if err := binary.Read(r, binary.BigEndian, &p.ElevationGain); err != nil {
    return nil, err
  }

  if err := binary.Read(r, binary.BigEndian, &p.ElevationLoss); err != nil {
    return nil, err
  }

  return &p, nil
}
```

This code looks horrible—even looking at it is causing me pain. This is
where monadic error handling comes to the rescue: just define a helper
structure that holds the actual io.Reader and the last error encountered
so far, and a read function that calls underlying Read only if previously
there were no errors. Here's the complete example:

```go
type reader struct {
  r   io.Reader
  err error
}

func (r *reader) read(data interface{}) {
  if r.err == nil {
    r.err = binary.Read(r.r, binary.BigEndian, data)
  }
}

func parse(input io.Reader) (*point, error) {
  var p point
  r := reader{r: input}

  r.read(&p.Longitude)
  r.read(&p.Latitude)
  r.read(&p.Distance)
  r.read(&p.ElevationGain)
  r.read(&p.ElevationLoss)

  if r.err != nil {
    return nil, r.err
  }

  return &p, nil
}
```

This is much better: main code path is no more obfuscated and you can
see at the first what the function is doing. The bad thing about this
technique is that you can't generalize it—you have to define a wrapper
type for each function. The good thing is that it's applicable to both
io.Reader and io.Writer, and lots of other read/write functions in the
standard library.

If you ever find yourself in situation where error handling is the main
part of your code, consider using this approach to make your functions
shorter and easier to understand.

## Add context to your errors

While I was working on a draft for this post, Coda Hale
[twitted](https://twitter.com/coda/status/859848060058296320)
something that is much better introduction to this section
than anything I could have written myself:

>"Go errors are great because you have to handle them but you
should wrap them and stack traces would be nice"
i too like checked exceptions

It's tempting to just pass errors to the callers in the following way:

```go
if err != nil {
  return err
}
```

But you don't want to see low-level errors in high-level functions:
what can the high-level caller do with "invalid syntax" error from
from strconv.Atoi call from the depths of the call stack?
That's why Go programmers are encouraged to either handle errors
or wrap them with additional details:

```go
if err != nil {
  return fmt.Errorf("something failed: %v", err)
}
```

This is better, and fine for most situations, but I don't like this
approach. The best solution for wrapping errors in year 2017 is to
use printf-style functions like animals? Fortunately, Dave Cheney
wrote better [errors](https://github.com/pkg/errors) package that
helps you add context to your errors in a better way:

```go
if err != nil {
  return errors.Wrap(err, "something failed")
}
```

You can also unwrap errors and add stack traces to them. It completely
eliminates the need for errors and fmt packages from the standard library,
because it also offers New and Errorf functions.

I really like this package and it's usually the first dependency I pull
for my new projects (I'll leave the discussion about whether it's worth
depending on another package for something simple like this for some
other time). In my opinion, this package should have been the part of
the standard library in the first place.
