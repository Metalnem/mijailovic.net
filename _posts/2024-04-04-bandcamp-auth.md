---
layout: post
title: "Reverse engineering Bandcamp authentication protocol"
image: /assets/img/2024-04-04-flowchart-preview.png
date: 2024-04-04 14:15:00 +0200
---
Did you know that the albums you purchase on Bandcamp can disappear from your collection
without notice? This can happen for various reasons. For example, a seller might
decide on a whim to remove the album from the platform. Bandcamp apparently
allows this in their [terms of use](https://bandcamp.com/terms_of_use):

> Content you purchase in a Transaction cannot be guaranteed to
be available to you perpetually.

> Users bear all risk from the denial of access to any Content
purchased through the Service.

The only way to make sure your albums stay in your possession is to download them
immediately after purchase. Heck, even Bandcamp officially recommends this:

> [...] we encourage you to promptly download any Content you purchase through the Site [...]

However, even if the album has been removed, and you hadn't dowloaded it, not all is lost.
In the Bandcamp mobile app, you can continue to listen to all your albums (but without the
option to download them), even after they've been removed from the platform. This obviously
means that Bandcamp doesn't delete the actual content from their servers. And if the app can
still access the lost albums, so can everyone who is patient enough to reverse engineer the app.
Surprisingly, no one has done this by now. Could it be that it's impossible? Let's dive in and
see what's going on inside the Bandcamp app!

## Inspecting the network traffic

As always, my first step was to inspect the network traffic between the Bandcamp
app and their backend servers. My favorite tool for this purpose has always been
[Burp Suite Community Edition](https://portswigger.net/burp/communitydownload).
After setting up the proxy and opening the collection page in the app, I quickly
noticed the following HTTP request in proxy logs:

```http
GET /api/collectionsync/1/collection?page_size=200 HTTP/2
Host: bandcamp.com
Authorization: Bearer MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3
```

This API endpoint returns the information about all your albums (things like album
name, band info, release date, purchase date, etc.). Not only that, but it also lists
all the album tracks, together with something that looks like high-quality audio URLs:

```json
{
  "token": "1:1700775127:355751800:a",
  "tralbum_id": 355751800,
  "title": "Pinnacle Of Bedlam",
  "tracks": [
    {
      "track_id": 3770803404,
      "title": "Cycles Of Suffering",
      "audio_url": "https://t4.bcbits.com/stream/b32687/mp3-128/3770803404",
      "hq_audio_url": "https://t4.bcbits.com/stream/fc3538/mp3-v0/3770803404",
      "track_number": 1
    },
    {
      "track_id": 2590214273,
      "title": "Purgatorical Punishment",
      "audio_url": "https://t4.bcbits.com/stream/2b6cad/mp3-128/2590214273",
      "hq_audio_url": "https://t4.bcbits.com/stream/a37aa9/mp3-v0/2590214273",
      "track_number": 2
    }
  ]
}
```

Unfortunately, `hq_audio_url` is a bit of a misnomer. High quality in this context
refers to MP3 V0, which is a lossy format (unlike regular Bandcamp downloads on the
web page, where you can choose from various selection of lossless formats). The good
news is that it's very unlikely you can even hear the difference between lossless
formats and high bitrate lossy formats. In any case, it's better to have your music
than not to have it at all, so I'll happily take MP3 V0 over nothing any day. Anyway,
I tried one of the download links from the JSON response and it worked:

![](/assets/img/2024-04-04-player.png)

First obstacle had been conquered. It was an important milestone for me, because
at this moment I knew that even if I couldn't figure out how to get the authentication
token programmatically, I would still be able to manually download the missing albums
from my collection: I could just save all HTTP responses and extract the audio URLs by
hand. It wouldn't be the most exciting job ever, but it would do the trick. Ah, who am
I kidding? Of course I would not be happy with such half-assed solution. I obviously
had to automate this process, which meant I needed to figure out how to get the
authentication token.

## Authentication protocol

Logins are typically very simple: you send a POST request with your username and
password, and you get an authentication token in return. Bandcamp's login protocol
is much more convoluted. Here is the high level description of the login flow:

1. App sends the login request to `/oauth_login` endpoint.
2. Server returns 418 status code and a hex-encoded, random-looking `X-Bandcamp-Dm` header.
3. App resends the login request with its own `X-Bandcamp-Dm` value.
4. Server returns 451 status code and introduces a new `X-Bandcamp-Pow` header.
5. App sends the final login request with its own `X-Bandcamp-Pow` value.
6. Server returns 200 status code and an authentication token.

![](/assets/img/2024-04-04-flowchart.png)

Based on this entire exchange, it appeared that `X-Bandcamp-Dm` and `X-Bandcamp-Pow`
response headers served as some sort of challenge. Correct outgoing header values were
necessary for successful authentication, and were in some way dependent on the incoming values.

Figuring out how the correct header values are generated just by looking at network traffic
was clearly impossible; the answer to this question could only be found in the client
application code. On the off chance that someone else had already figured out
the algorithm, I did a quick Google and GitHub search for `X-Bandcamp-Dm`. I got a
couple of hits, but all of them were just documenting the struggles of other people:

>ok how they handle the new DM is, basically put, a pain in the ass,
we'll see if I get around figuring out wtf is happening
([MITM Android Bandcamp app](https://github.com/the-eater/camp-collective/issues/2))

>I’d love to support that if someone wants to reverse-engineer
the X-Bandcamp-DM and X-Bandcamp-PoW headers.
([Mopidy-Bandcamp](https://pypi.org/project/Mopidy-Bandcamp/1.0.1/))

>Unfortunately I cannot recreate x-bandcamp-dm value in headers.
([Python Bandcamp scraper](https://github.com/milicamilivojevic/bandcamp))

Apparently, `X-Bandcamp-Dm` and `X-Bandcamp-Pow` headers were the secret ingredient
that made it difficult to reverse engineer the login API. It was time to decompile the mobile
app and find the answer to how these secret values are generated.

## Decompiling the Android app

Since reversing managed code is much easier than reversing native code, I chose
to decompile the Android mobile app. I've always used
[JADX](https://github.com/skylot/jadx) for this purpose and it has always served me well
(its [GUI features](https://github.com/skylot/jadx/wiki/jadx-gui-features-overview)
would turn out to be particularly useful). After downloading the Bandcamp application
package from [APKMirror](https://www.apkmirror.com/apk/bandcamp-inc/bandcamp/) and
opening it with JADX, I found out that the app was obfuscated:

![](/assets/img/2024-04-04-jadx.png)

I had never reverse engineered obfuscated code before. To determine how difficult the
process would be, I searched for all occurrences of string `X-Bandcamp-Dm`. Number of
results: zero. So, not only was the code obfuscated, but the string values were obfuscated
as well. That meant the job of figuring out how the mysterious header values were calculated
was not going to be easy. In fact, I wasn't sure if it was going to be possible at all,
since I didn't have any clue where to start. I had doubts whether I even wanted to
embark on this journey, but ultimately, I decided to do it, even if it takes me half a
year (luckily, I only needed three weeks).

## Obfuscation techniques

The app was using many different obfuscation techniques, and since I was a complete
newbie in this area, I had to learn from scratch how to defeat each one. I'm going
to show you some of the techniques I've seen, along with the tips on how to fight
them. Reverse engineering veterans among you probably know all of them already, but
if you are a beginner, I hope that you'll learn something new and see that obfuscation
is not as intimidating as it might appear.

### Renaming

Renaming is an obfuscating method where identifiers (variable, class, field, and
method names) are renamed to random gibberish. This is probably the most well-known
type of obfuscation, so it's not surprising that it's frequently used in Bandcamp
mobile app. For example, a typical method call might look something like this:

```java
dVar.F(f17618q, b(dVar.i()));
```

When I first looked at this code, I had no idea what methods `F`, `b` and `i` were doing.
Luckily, JADX is almost a full-blown IDE, so it contains features such as "Find usage"
and "Go to declaration". These two made the analysis much easier, because I was able to
traverse the call chains until I reached some method with a normal, unobfuscated
name, such as this one:

```java
public void setHeaders(s7.d dVar) {
  this.f4578a.d(dVar);
}
```

When you repeat this process for all unknown method names, you will eventually discover
that `F` sets the request header value in the HTTP client, `b` calculates the value of
the header, and `i` composes the body of the outgoing HTTP request. The obfuscated code
then becomes something that you can easily reason about:

```java
request.setHeaders("X-Bandcamp-Dm", calculateHash(request.getParams()));
```

After spending a lot of time with the obfuscated code, you become so familiar
with it that you start noticing things that were impossible to see before. For
example, after a while it became obvious to me that these two methods were aliases
for HMAC SHA-256 and HMAC SHA-512 cryptographic hash functions, respectively:

```java
public static String e(String str, String str2, int i10, float f10) {
  return com.bandcamp.shared.platform.a.d().h(str, null, str2, i10, f10);
}

public static String d(String str, String str2, int i10) {
  return com.bandcamp.shared.platform.a.d().B(str, null, str2, i10);
}
```

Once you discover the real purpose of a method or a variable, you can also rename it
in JADX. I didn't use this feature, though. After finally understanding the meaning
of the code, I didn't feel the need to rename anything, because I had already formed a
mental map, and the obfuscated code began to look just like regular code to me. This
would probably be more challenging for larger apps or if I had to deobfuscate multiple
features, not just header calculation.

### String obfuscation

Even when all method and variable names are random nonsense, you still expect to at
least be able to search for string constants. Since the app sends and receives the
header `X-Bandcamp-Dm`, that string has to be somewhere in the code, right? But as
I mentioned earlier, that header name was nowhere to be found. How do you even proceed
from here? I started looking for string fragments. How about the string `"X"`, the
first character of the header name? There were dozens of occurrences of this value
across the codebase, and most of them led nowhere, but there was also this one:

```java
public static String f17618q = "X";
public static String f17620s = "pmac";
public static String f17621t = "D";
public static String f17622u = "M";
public static String f17619r = "nab";

static {
  f17620s += "d";
}

public <T> void d(s7.d<T> dVar) {
  char[] charArray = f17619r.toCharArray();
  for (int i10 = 0; i10 < charArray.length / 2; i10++) {
    char c10 = charArray[i10];
    charArray[i10] = charArray[(charArray.length - i10) - 1];
    charArray[(charArray.length - i10) - 1] = c10;
  }
  String str = new String(charArray);
  char[] charArray2 = f17620s.toCharArray();
  for (int i11 = 0; i11 < charArray2.length / 2; i11++) {
    char c11 = charArray2[i11];
    charArray2[i11] = charArray2[(charArray2.length - i11) - 1];
    charArray2[(charArray2.length - i11) - 1] = c11;
  }
  String str2 = new String(charArray2);
  dVar.F(f17618q + "-" + str + str2 + "-" + f17621t + f17622u, b(dVar.i()));
}
```

`X`, `pmac`, `D`, `M`, `nab`, `d`—what a weird bunch. Hm, but doesn't it look an awful
lot like something we are looking for? If your first thought was "this seems to
be a permutation of the `X-Bandcamp-Dm` header name", you were 100% right! The
sole purpose of this class is to hide the well-known string by constructing it
using string concatenation and reversing. All these shenanigans can be replaced
with a single line of code:

```java
public <T> void d(s7.d<T> dVar) {
  dVar.F("X-Bandcamp-Dm", b(dVar.i()));
}
```

Searching for string fragments has served me well multiple times, so I hereby
officially declare it to be a very useful method for finding obfuscated string
values.

### Reflection

This is where analyzing the obfuscated code becomes much more difficult. I've
mentioned earlier that even if a method has been renamed, you can still learn
something about it by following its call chain. But in some cases, you don't
have this luxury: if a method is invoked using reflection, you can't track
its usage directly anymore. Take a look at this simple example:

```java
obj.getClass()
  .getMethod("v0".replace("0", "alue"), new Class[0]);
  .invoke(obj, new Object[0]);
```

If you searched for all usages of method `value()` defined in the 
`CacheListenerEvent` class, you wouldn't have found anything. But if you
knew that it might have been called via reflection, you could have searched
for `value`, `val`, or `lue`, and you would have found this call eventually.
It's not always that easy, though. In some cases, even string search wouldn't
have helped you:

```java
Class.forName(sb2.toString())
  .getMethod(x7.d.c("lmrgdw", 2), Object.class)
  .invoke(cls, "2" + obj.toString().replaceAll("3", "5"));
```

Which method is being called here? You can't easily discover that using only
static analysis—you must directly invoke `x7.d.c("lmrgdw", 2)` to determine
the result of the call.

In the end, there is no guaranteed way to defeat this obfuscation technique. You
just need to be patient, and in the worst case, be ready to search for all usages
of reflection to find that single call you need.

## X-Bandcamp-Dm

I showed you all these obfuscation techniques because all of them were used
in some form in the `X-Bandcamp-Dm` calculation. As I was learning more about
deobfuscation, I was also slowly piecing together the `X-Bandcamp-Dm` algorithm.
One day, I would learn how the header name was being constructed and where it was
used. The next day, I would learn that the final header value is an output of the
HMAC function and what its inputs are. After that, I would reverse engineer the
weird, home-made key derivation function used to generate the keys for the HMAC
calculation. Ultimately, I realized that `X-Bandcamp-Dm` is an HMAC SHA-256
hash of the incoming `X-Bandcamp-Dm` value, outgoing HTTP request body, and one
more value that I couldn't yet identify. That unidentified value was being
initialized in the following method:

```java
@Override // java.util.Observer
public void update(Observable observable, Object obj) {
  if ((obj instanceof String) && f17625n + 48 == ((String) obj).charAt(0)) {
    f17624m = obj.toString().substring(1).getBytes("utf-8");
  }
}
```

Of course, there was a catch—I couldn't find any direct callers of this method.
It could mean only one thing: the call sites were obfuscated to use reflection.
I tried brute-forcing my way out of this problem by inspecting every observer chain
in the code, but there were hundreds of them. Most of them were obfuscated, so this
approach wasn't going to work in a reasonable timeframe.

I was stuck on this for almost one entire week. After many unsuccessful attempts to find
the caller of this method, it finally dawned on me. The condition before the assignment
was checking if the string `obj` starts with the character `"3"` (the value of the
field `f17625n` was always 3, and 48 is the numeric value of ASCII character 0). This
meant that `obj` didn't start with 3 randomly, but on purpose. Otherwise, the code
couldn't possibly work. And what's the way to ensure that something starts with 3?
Well, prepend `"3"` to it, of course! I searched for `"3" +` and found this:

```java
@Override // java.util.Observer
public void update(Observable observable, Object obj) {
  ((Class) ((Object[]) obj)[2]).getMethod(
    a.this.f18941o.substring(0, 5) + a.this.f18942p.substring(0, 1),
    Object.class
  ).invoke(
    (Class) ((Object[]) obj)[2],
    "3" + x7.h.d(
      (String) ((Object[]) obj)[0],
      (String) ((Object[]) obj)[1],
      0
    )
  );
}
```

This was the most heavily obfuscated piece of code I had encountered.
It was similar to a final boss fight, because it was using all obfuscation methods
that had been bothering me previously (renaming, string obfuscation, reflection).
However, by this point, all of this had become standard procedure for me. I quickly
discovered that the reflection call was invoking the method `notify`, which was then
notifying the observer I was interested in. This was the final piece of the puzzle!
I deobfuscated the remaining parameters and updated my API client code. The moment
I ran it and finally received the HTTP status code 451 instead of 418 from Bandcamp
servers will forever remain as one of the happiest moments in my hacking history.

Looking back, it's so funny that the `X-Bandcamp-Dm` calculation algorithm is so simple
and clean and so easy to describe, yet it took me weeks to recreate it from thousands of
lines of obfuscated code.

```csharp
var input = response.Headers["X-Bandcamp-Dm"];

var key1 = FunkyKdf1(input, staticKey1);
var key2 = FunkyKdf2(input, staticKey2);

var output = HmacSha256(key2 + request.Body, key1);

request.Headers["X-Bandcamp-Dm"] = output;
```

## X-Bandcamp-Pow

With `X-Bandcamp-Dm` out of the way, it was time to figure out the meaning of the
`X-Bandcamp-Pow` header. Compared to the time I had spent on `X-Bandcamp-Dm`, reversing
the calculation of `X-Bandcamp-Pow` was a breeze. It turned out to be a proof-of-work
scheme that closely resembles [Hashcash](http://www.hashcash.org/), the scheme that
inspired Bitcoin's own proof-of-work implementation. Bandcamp's version concatenates
the request body with the incoming `X-Bandcamp-Pow` value and an increasing counter.
Next, it repeatedly calculates the SHA-1 hash of the new string until the output has the
desired number of leading zero bits. The final counter value is then encoded using Base36
and appended to the original `X-Bandcamp-Pow` value.

For example, if `X-Bandcamp-Pow` is `1:10:f6e592b662b3`, it means we need to find a
hash with 10 leading zero bits. If we find it in 760 iterations, then the outgoing
`X-Bandcamp-Pow` value will be `1:10:f6e592b662b3:l4` (760 is l4 in Base36).

It seems to me that the only reason for the introduction of this header was that
everyone wanted to be a part of the blockchain craze at that time (`X-Bandcamp-Pow`
was first introduced in December 2019, a year and a half after `X-Bandcamp-Dm`).
I don't see any other explanation, because `X-Bandcamp-Pow` doesn't offer any
additional advantages over `X-Bandcamp-Dm` (which can't be brute-forced anyway).

But I digress. The moment of truth had arrived. I implemented proof-of-work
calculation in my API client, ran it, and got the following output:

```
HTTP/2 418 I'm a teapot
HTTP/2 451 Unavailable For Legal Reasons
HTTP/2 200 OK
```

My first login request was successful, and the authentication token was finally
mine! After this, implementing the rest of the API for downloading the albums
from the collection was trivial.

## Bandcamp downloader

The command line tool I wrote is available
[here](https://github.com/Metalnem/bandcamp-downloader).
It has an absolutely minimal set of features: you can list all your purchased
albums and you can download a specific album from your collection in MP3 V0
format. Here is one usage example:

```shell
# List all albums in your Bandcamp collection
$ dotnet run --username $USERNAME --password $PASSWORD
870109722 Bolt Thrower — Realm of Chaos
910230745 Cannibal Corpse — Evisceration Plague
157725502 Cryptopsy — None So Vile
388372040 Incantation — Onward to Golgotha
212824804 Archspire — Relentless Mutation

# Download the album with the specified ID
$ dotnet run --username $USERNAME --password $PASSWORD --album 870109722
```

I don't plan to extend it with more features, since my main goal in this quest was
to enable Bandcamp users to download the albums they can't download in any other way.
Also, there are already many feature-rich Bandcamp downloaders around, and it would
make more sense to extend them with proper authentication than to reimplement all their
features from scratch in my repo. If you are a maintainer of one such downloader, feel
free to reuse the authentication code that I have implemented.

Enjoy downloading your lost albums and listening to them once again!
