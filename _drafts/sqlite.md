---
layout: post
title: "Adventures with SQLite and SQLITE_OPEN_EXCLUSIVE"
---
Proper online backup solutions such as [Tarsnap](https://www.tarsnap.com/)
create snapshots of your data, which means you can never destroy your
backup by accident. If you have something that you can't afford
to lose, Tarsnap is the only online backup solution that I could
recommend. However, Tarsnap can get very expensive for the large
datasets. It would be insane to use it for hundreds of gigabytes
of music, for example. Dropbox is a much better choice in such case,
but it's not a really a backup solution. You could accidentally
delete or modify some file, and notice only when it's too late
to recover it.

That is why I recently implemented a small integrity checking
[tool](https://github.com/Metalnem/hashes) for my backup folders.
It calculates checksums of all files in specified directories and
stores them in an immutable SQLite database (each time the tool
is invoked it creates the new file instead of keeping everything
in a single database). It can also calculate the difference between
the two databases, therefore allowing me to verify that no files
were modified or removed in the meantime. I could have used
a simple file instead of an SQLite database, but SQLite is almost
always a better choice for an application file format (read
[this](https://sqlite.org/appfileformat.html) to see why). Also,
file consistency is [hard](https://danluu.com/file-consistency/).

In order to prevent my program from accidentally overwriting the
data in some of the previous databases, I wanted to satisfy two
safety related requirements:

- When comparing the two databases, I wanted to open them in
read-only mode, thus making it impossible to destroy anything.
- When creating the new database, I wanted to always create
the new file, and fail if the file with the same name already
exists. Again, this makes doing destructive changes impossible.

First requirement can easily be satisfied with both ordinary
files and SQLite databases. Second requirement is often source
of security problems with ordinary files, but it is also easy
to satisfy. With SQLite, however, situation is very different,
and that’s the topic of my blog post.

## Safely creating a new file

First, let’s talk a little about opening files. I will focus
on the Unix system call [open](http://man7.org/linux/man-pages/man2/open.2.html),
but the general idea is also applicable to Windows (see the
[CreateFile](https://msdn.microsoft.com/en-us/library/windows/desktop/aa363858.aspx)
documentation for more details).

When you want to open or create a file on Unix, whether it’s
for reading, writing, or both, you use the **open** system
call. Its signature looks like this:

```c
int open(const char *pathname, int flags);
```

Here are the two most common ways of using the **open** function:

```c
// Open the file in read-only mode.
int fd1 = open("file1", O_RDONLY);

// Open the file for both reading and writing.
// Create it if it doesn't exist.
int fd2 = open("file2", O_RDWR | O_CREAT);
```

First call is fine, but the second call could be problematic.
If the file already exists before this call, it will be opened.
This is sometimes source of symbolic link attacks, where instead
of opening a new file, attacker can trick you into overwriting
some other, often security critical file. But that was not my
main concern; I was more worried about the situation where I
could overwrite some old database, only because I named the
new one with the same name by accident. Is there a way to
create a file only if it does not exist? Of course—that is
exactly what the **O_EXCL** flag is for:

>Ensure that this call creates the file: if this flag is
specified in conjunction with **O_CREAT**, and *pathname*
already exists, then **open()** will fail.

This does the trick for plain files. Can we do something
similar when working with SQLite databases?

## Safely creating a new SQLite database

SQLite databases are opened using the **sqlite3_open_v2**
function (you could also use the **sqlite3_open** or
**sqlite3_open16** [functions](https://sqlite.org/c3ref/open.html),
but they are not as powerful as **sqlite3_open_v2**, so
we can ignore them in this discussion):

```c
int sqlite3_open_v2(const char *filename, sqlite3 **ppDb, int flags, const char *zVfs);
```

It looks very similar to the **open** function, which should
not be surprising, because SQLite developers clearly state
that SQLite databases should be used instead of ad-hoc
files in most situations. SQLite uses its own set of
[flags](https://sqlite.org/c3ref/c_open_autoproxy.html), but
they also look very similar to their **open** counterparts.
We are interested in only four of them at this moment:

```c
#define SQLITE_OPEN_READONLY         0x00000001  /* Ok for sqlite3_open_v2() */
#define SQLITE_OPEN_READWRITE        0x00000002  /* Ok for sqlite3_open_v2() */
#define SQLITE_OPEN_CREATE           0x00000004  /* Ok for sqlite3_open_v2() */
#define SQLITE_OPEN_EXCLUSIVE        0x00000010  /* VFS only */
```

This looks like the *O_RDONLY*, *O_RDWR*, *O_CREAT*,
and *O_EXCL* flags have their direct equivalents (people
often think that *SQLITE_OPEN_EXCLUSIVE* opens the
database for exclusive access, but that is not what
it does). But why are the first three flags compatible
with the **sqlite3_open_v2** call, but *SQLITE_OPEN_EXCLUSIVE*
is not? Why the comment says it’s “VFS only”, and
what is VFS, actually?

[VFS](https://sqlite.org/vfs.html) stands out for
“Virtual File System”, and it's a portability layer
for abstracting the file system operations across
different operating systems. SQLite ships with
multiple Unix and Windows implementations, but you
can even write your own VFS if you want. But
this doesn’t answer the our question—what does it mean
that the flag we are interested is marked as VFS only?
Let’s write a simple program to see what happens when
we use the *SQLITE_OPEN_EXCLUSIVE* flag.

```c
#include "sqlite3.h"
#include "stdio.h"
#include "stdlib.h"

int main()
{
  sqlite3 *db;
  char *sql, *err;
  int flags, rc;

  flags = SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_EXCLUSIVE;

  if ((rc = sqlite3_open_v2("test.db", &db, flags, NULL)) > 0)
  {
    fprintf(stderr, "%s\n", sqlite3_errstr(rc));
    exit(1);
  }

  sql = "CREATE TABLE IF NOT EXISTS foo(value); \
    INSERT INTO foo(value) VALUES('value');";

  if ((rc = sqlite3_exec(db, sql, NULL, NULL, &err)) > 0)
  {
    fprintf(stderr, "%s\n", err);
    exit(1);
  }

  return 0;
}
```

This program should open the database for reading and writing,
and fail if it already exists. But that is not what happens:
it completes without any errors when we run it. *SQLITE_OPEN_EXCLUSIVE*
has clearly been ignored, which we could have suspected
from its description. To find out what is going on, we
have to dig deeper.

## Analyzing the SQLite source code

The easiest way to analyze and compile the SQLite
source is to download so called
[amalgamation](https://www.sqlite.org/amalgamation.html).
It is a single file called "sqlite3.c", which is just a
concatenation of all SQLite source files. That makes
compilation, navigation, and searching much easier.

The first function from the SQLite library that we
are calling in our sample program is **sqlite3_open_v2**,
so let’s find its definition, and follow the propagation
of the flags from there. Its definition looks like this:

```c
int sqlite3_open_v2(
  const char *filename,   /* Database filename (UTF-8) */
  sqlite3 **ppDb,         /* OUT: SQLite db handle */
  int flags,              /* Flags */
  const char *zVfs        /* Name of VFS module to use */
){
  return openDatabase(filename, ppDb, (unsigned int)flags, zVfs);
}
```

It's just a simple wrapper around the **openDatabase**,
so let's continue our search there. Immediatelly at the top
of the **openDatabase** is the answer to our question
why the *SQLITE_OPEN_EXCLUSIVE* flag is being ignored:

```c
/* Remove harmful bits from the flags parameter
**
** The SQLITE_OPEN_NOMUTEX and SQLITE_OPEN_FULLMUTEX flags were
** dealt with in the previous code block. Besides these, the only
** valid input flags for sqlite3_open_v2() are SQLITE_OPEN_READONLY,
** SQLITE_OPEN_READWRITE, SQLITE_OPEN_CREATE, SQLITE_OPEN_SHAREDCACHE,
** SQLITE_OPEN_PRIVATECACHE, and some reserved bits. Silently mask
** off all other flags.
*/
flags &=  ~( SQLITE_OPEN_DELETEONCLOSE |
             SQLITE_OPEN_EXCLUSIVE |
             SQLITE_OPEN_MAIN_DB |
             SQLITE_OPEN_TEMP_DB | 
             SQLITE_OPEN_TRANSIENT_DB | 
             SQLITE_OPEN_MAIN_JOURNAL | 
             SQLITE_OPEN_TEMP_JOURNAL | 
             SQLITE_OPEN_SUBJOURNAL | 
             SQLITE_OPEN_MASTER_JOURNAL |
             SQLITE_OPEN_NOMUTEX |
             SQLITE_OPEN_FULLMUTEX |
             SQLITE_OPEN_WAL
           );
```

This code is being executed before any other useful action
takes place (even before the VFS is instantiated). We now
know that our flag is being silently masked off, but the
code still doesn't explain why the flag is not valid.
Another comment in the same function says which
combinations of flags are allowed:

```c
/* Parse the filename/URI argument
**
** Only allow sensible combinations of bits in the flags argument.  
** Throw an error if any non-sense combination is used. If we
** do not block illegal combinations here, it could trigger
** assert() statements in deeper layers. Sensible combinations
** are:
**
**  1:  SQLITE_OPEN_READONLY
**  2:  SQLITE_OPEN_READWRITE
**  6:  SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE
*/
```

Again, this is not really an explanation. Why is the
combination of *SQLITE_OPEN_READWRITE*, *SQLITE_OPEN_CREATE*,
and *SQLITE_OPEN_EXCLUSIVE* missing from the list of sensible
combinations, when it's actually pretty sensible? It turns out
I'm not the only one who's wondering about that—other people were
[asking](http://sqlite.1065341.n5.nabble.com/SQLITE-OPEN-EXCLUSIVE-td81156.html)
the same question, too. I found the closest thing to a real
explanation on the SQLite mailing list post called
[sqlite3_open() exclusive?](http://sqlite-users.sqlite.narkive.com/8HoxFb5I/sqlite3-open-exclusive),
where the author of the library says:

>SQLite databases are designed to be shared by two or more
processes, so no it does not use O_EXCL.

This is a sane default choice, but it still doesn't
say why the flag is explicitly forbidden.

We have concluded that the *SQLITE_OPEN_EXCLUSIVE* flag is
definitely not allowed, but we still don't know what does
it mean that it's VFS only, so let's learn more about that.

Function **openDatabase** doesn't open the database
file directly—it actually opens the file using the
function provided by the VFS implementation for the
current operating system. Files on the Unix operating
systems are opened using the **unixOpen** function:

```c
static int unixOpen(
  sqlite3_vfs *pVfs,           /* The VFS for which this is the xOpen method */
  const char *zPath,           /* Pathname of file to be opened */
  sqlite3_file *pFile,         /* The file descriptor to be filled in */
  int flags,                   /* Input flags to control the opening */
  int *pOutFlags               /* Output flags returned to SQLite core */
)
```

Unlike the **sqlite3_open_v2**, **unixOpen** does handle
*SQLITE_OPEN_EXCLUSIVE* flag correctly. And not just that,
but the flag is actually used internally in several
places. For example, temporary databases are created
using this flag. It's only external users of SQLite
who are being prevented from using the flag.

So, where does that leaves us? We could modify the
SQLite source code to not mask the flag, but that's
not really an acceptable solution to our problem.
Can we do better than that?

## Writing a custom VFS

I was ready to give up at this point, because by this
time I already lost countless hours on what was really
a minor problem (if you could call it a problem at all).
But for some reason I couldn't leave the job unfinished,
so I decided it was time for the nuclear option: writing
a custom VFS implementation.

Implementing the VFS requires overriding the **sqlite3_vfs**
[structure](https://sqlite.org/c3ref/vfs.html). It
contains more than a dozen functions and several fields.
Default implementations for Unix and Windows are
huge and complicated; doing something like that
from scratch would be an enormous and error-prone
job.

When you think more about it, it's not actually
necessary to implement all these functions from
scratch. Almost all of them should just be reused;
the only one that we really have to override is
the **xOpen** function:

```c
int *xOpen(sqlite3_vfs*, const char *zName, sqlite3_file*, int flags, int *pOutFlags);
```

Even this function doesn't have to be implemented
from scratch—we only have to reapply the
*SQLITE_OPEN_EXCLUSIVE* flag, and then call
the original **xOpen** function, because we
know that it's capable of handling it.

So, the solution is to just write a small wrapper
around the default VFS that would just forward
the calls to the original functions, but also
add the *SQLITE_OPEN_EXLUSIVE* flag in the **xOpen**
function before calling the real version.
SQLite documentation was helpful because it contains the
[implementation] (http://www.sqlite.org/src/doc/trunk/src/test_vfstrace.c)
of VFS shim that writes diagnostic output for each VFS call.

I wanted to apply the flag only when *SQLITE_OPEN_READWRITE*
and *SQLITE_OPEN_CREATE* flags were already applied, so my
initial attempt to override the **xOpen** function looked
something like this:

```c
int xOpen(sqlite3_vfs *vfs, const char *name, sqlite3_file *file, int flags, int *outFlags)
{
  sqlite3_vfs *root = (sqlite3_vfs *)vfs->pAppData;

  if (flags & (SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE))
  {
    flags |= SQLITE_OPEN_EXCLUSIVE;
  }

  return root->xOpen(root, name, file, flags, outFlags);
}
```

This kinda worked, but was not really the correct solution.
The problem with this code is that the **xOpen** function
was not used only for creating the database: SQLite is also
using it internally for other purposes, like creating
the journal, write-ahead log, and many other temporary files.
I could not guarantee that adding the *SQLITE_OPEN_EXLUSIVE*
flag wouldn't break some of the internal operations.

Is there a way to add the flag only when we are opening the
main database file? If your answer is yes, you are correct:
there is an additional flag called *SQLITE_OPEN_MAIN_DB*.
SQLite applies an additional flag for each type of file
it creates, and the main database is one of them (you can
see the list of all flags
[here](https://sqlite.org/c3ref/c_open_autoproxy.html).
Armed with this knowledge, I just had to slightly modify
my previous attempt:

```c
int xOpen(sqlite3_vfs *vfs, const char *name, sqlite3_file *file, int flags, int *outFlags)
{
  sqlite3_vfs *root = (sqlite3_vfs *)vfs->pAppData;

  if (flags & (SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_MAIN_DB))
  {
    flags |= SQLITE_OPEN_EXCLUSIVE;
  }

  return root->xOpen(root, name, file, flags, outFlags);
}
```

As far as I'm concerned, this looks like correct and
portable solution. Also, you should know that even
with this custom VFS, SQLite will still open the
database, but in read-only mode (default VFS implememtations
have a fallback to opening the database in read-only
mode if the main open call fails for any reason).
That's fine, though: we only wanted to be sure
that we won't overwrite any data by accident,
and opening the database in read-only mode satisfies
that requirement.

## Conclusion

Opening an SQLite database safely should have been
a very simple thing to do, but it turned into a
week-long pointless and fun exercise. You can find
the complete implementation of my custom VFS
[here](https://github.com/Metalnem/sqlite-vfsdemo).
It is just a proof of concept, so be careful if
you want to use in the real-world code (but you
probably don't need it, anyway).