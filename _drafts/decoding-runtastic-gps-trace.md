---
layout: post
title: "Decoding Runtastic GPS trace"
---
# Decoding Runtastic GPS trace

I've written about reversing Runtastic API in my [previous post]({{ site.baseurl }}{% post_url 2016-10-20-reversing-runtastic-api %}).
As you might remember, one part of the activity data returned from the server was the GPS trace field. I could have tried to decode it
and extract the activity data, but I chose to ignore it back then because there
was a much easier way to export the activity. That changed few days after the Runtastic Archiver application was completed. I was running
the application to backup my latest runs, when I suddenly started getting errors that some expected headers in HTTP responses were missing.
It turned out that server introduced some kind of rate limiting for export requests. First activity was successfully downloaded every time, but
the next one always failed. After experimenting with delays between requests, I discovered that the Runtastic server was allowing only one
request every 10 seconds. Delay that long was unacceptable. For example, if you run 5 times per week, and you were using Runtastic
for the last 4 years, you would have around 1000 activities, and your backup process would be almost 3 hours long. Web application was no
exception to the rate limiting - if you tried to export multiple activities in a short period of time, all files, with the exception
of the first one, would contain only **access denied** string. At that moment I had no other choice but to get back to encoded GPS trace
field, decode it and
then manually create gpx export file.

## Decoding the trace, part one

My first step was to create one simple activity with just a few GPS points, which I hoped would make the decoding process easier.
I did that by exporting one of my activities and removing all info except the first three GPS data points. The final gpx file that I decided
to use looked like this (I removed the XML namespaces for simplicity):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gpx>
  <metadata>
    <time>2016-09-09T16:00:52.000Z</time>
  </metadata>
  <trk>
    <trkseg>
      <trkpt lon="20.4706268310546875" lat="44.8097534179687500">
        <ele>129.48464965820312</ele>
        <time>2016-09-09T16:00:53.000Z</time>
      </trkpt>
      <trkpt lon="20.4706439971923828" lat="44.8096694946289062">
        <ele>129.56248474121094</ele>
        <time>2016-09-09T16:00:54.000Z</time>
      </trkpt>
      <trkpt lon="20.4706783294677734" lat="44.8095474243164062">
        <ele>129.6656036376953</ele>
        <time>2016-09-09T16:00:57.000Z</time>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

After creating the file, I uploaded it to Runtastic server using the web application. I started the Runtastic iPhone app with the Burp
Suite intercepting the network traffic, waiting for the app to synchronize with the server. I recorded the following GPS trace:

```
AAAAAwAAAVcPriMIQaPD2EIzPTBDAXwSAAAAAAAAAAAAAAAAAAAAAAAAAAABVw+uJvBBo8PhQjM9
GkMBj/8AAEIHyUEAAAPoAAAACQABAAAAAAFXD64yqEGjw/NCMzz6QwGqZQAAQYTgDwAAD6AAAAAX
AAEAAA==
```

This seemed promising. I feared that GPS trace might be encrypted, but this string had a lot of redundancy in it (enrypted data would
be indistinguishable from random data). Most of it were just alphanumeric characters - the rest were +, / and =. That suggested
[Base64](https://en.wikipedia.org/wiki/Base64) encoding. After decoding the string I got an array of bytes looking like this:

```
0 0 0 3 0 1 87 15 174 35 8 65 163 195 216 66 51 61 48 67 1 124 18 ...
```

First non-zero byte was 3. That was also the number of data points in the activity, so I assumed that the trace had the length prefix
in [big-endian](https://en.wikipedia.org/wiki/Endianness) format, followed by the actual encoding of every data point. After removing
the first 4 bytes (length of one 32-bit integer), I tried to split the remaining ones into three equal parts (and also convert them
to their hexadecimal representation). The result looked like this:

```
00 00 00 03
00 00 01 57 0f ae 23 08 41 a3 c3 d8 42 33 3d 30 43 01 7c 12 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 01 57 0f ae 26 f0 41 a3 c3 e1 42 33 3d 1a 43 01 8f ff 00 00 42 07 c9 41 00 00 03 e8 00 00 00 09 00 01 00 00
00 00 01 57 0f ae 32 a8 41 a3 c3 f3 42 33 3c fa 43 01 aa 65 00 00 41 84 e0 0f 00 00 0f a0 00 00 00 17 00 01 00 00
```

This looked nice! Every row started with the same 6 bytes, and bunch of bytes on other positions were same for all of the rows,
which confirmed my theory that every row was the encoded data for single GPS point. Now I had to discover the meaning of each byte.

Exported gpx file contains longitude, latitude, elevation and time for each data point. My first idea was to convert those values to hex
and then try to find them in the trace. Because I was super lazy, I used [this](https://gregstoll.dyndns.org/~gregstoll/floattohex/)
website to do the floating-point conversions. There are two types of floating-point numbers - single precision and double precision
(or simply float and double). I didn't know which one the application was using, so I tested both. Converting longitude from the first
data point (20.4706268310546875) was 0x41a3c3d8 when represented as a float, and 0x4034787b00000000 when represented as a double.
Float version matched perfectly with the bytes 8, 9, 10 and 11, so I kept using the float representation for the remaining values.
Hex representation of the latitude was 0x42333d30, which matched bytes 12, 13, 14 and 15. Elevation was 0x43017c12, found in bytes
16, 17, 18 and 19.

Time was slightly different. There are various ways to serialize date and time, but the most common one is
[Unix time](https://en.wikipedia.org/wiki/Unix_time). I used [Epoch Unix Time Stamp Converter](http://www.unixtimestamp.com/index.php)
for converting time found in gpx file to Unix time. Time from the first data point was converted to 1473436853, which is 0x57d2dcb5 in hex.
But that value was nowhere to be found in the trace. Because I still haven't decoded the first 8 bytes, I tried to read them as one
64-bit integer. The result I got was number 1473436853000, which was the Unix time I needed. The only difference from standard
representation was that it was actually number of milliseconds instead of seconds.

At this moment I had everything I needed to export the activity. I modified my original program to decode the values I needed and to
manually create the gpx file. Runtastic web application successfully accepted the file when I tried to upload it, so everything was good.
But I couldn't stop there, I had to know what were the remaining bytes.

## Decoding the trace, part two

The remaining unknown bytes looked like this:

```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 42 07 c9 41 00 00 03 e8 00 00 00 09 00 01 00 00
00 00 41 84 e0 0f 00 00 0f a0 00 00 00 17 00 01 00 00
```

I couldn't conclude anything from looking at those values, so I decided to look at the bigger picture. I downloaded one of my larger runs and
extracted first and last five rows of the unknown data:

```
08 05 00 00 00 00 00 00 06 f5 00 00 00 00 00 00 00 00
08 05 41 20 b4 39 00 00 0a 4c 00 00 00 0a 00 00 00 00
08 05 41 33 39 46 00 00 12 1c 00 00 00 12 00 00 00 00
08 05 41 47 bc 08 00 00 19 ec 00 00 00 1c 00 00 00 00
08 05 41 52 f3 ab 00 00 25 df 00 00 00 2a 00 00 00 00
...
03 05 41 3c 48 93 00 2e bb 86 00 00 27 12 00 00 00 00
03 05 41 3e d8 80 00 2e c7 11 00 00 27 1d 00 00 00 00
03 05 41 40 e1 af 00 2e ce e1 00 00 27 24 00 00 00 00
03 05 41 50 cb b7 00 2e d6 ae 00 00 27 2c 00 00 00 00
03 05 41 50 cb b7 00 2e e4 02 00 00 27 2c 00 00 00 00
```

This looked much more promising, because I could see some interesting patterns here. First two bytes were the same almost all the time,
so I ignored them at first. After them came four groups of four bytes each. First group looked like floating-point numbers, second and
third group looked like monotonically increasing integers, and fourth group was always zero. There are only a few possible values that
are always increasing during the run - total time and total distance. If some of the unknown values were really total time or total
distance, final row should contain final distance and duration of the run. Those values for my run looked like this:

```
Distance: 10.03km
Duration: 00:51:13
Pace: 05:06 min/km
Average Speed: 11.75 km/h
Max. Speed: 14.47 km/h
Total Steps: 9040
```

After converting first group of bytes (0x4150cbb7) to float, the result was number 13.0497. That value looked like speed - it was below
the maximum and above the average speed. To confirm that value was really the speed, I converted all the values in the file and found
the maximum one - it was indeed 14.47.

Second group (0x002ee402) was 3073026 in decimal. That didn't look very familiar. It certainly didn't look like total distance, so maybe
it was total duration expressed in some different way? 51:13 is exactly 3073 seconds, so the second group of bytes was total duration
of the run in milliseconds.

Third group (0x0000272c) was easy - it was 10028 in decimal, which was total distance in meters.

Fourth group was always zero, which was not very useful information. If was actually zero for the most of my runs. Just few of them had
some non-zero values. I downloaded the trace for one of them, and the final three data points had these values for the last four bytes:

```
00 47 00 42
00 47 00 42
00 47 00 42
```

That didn't look like one 32-bit integer to me. It was more probable that there were actually two 16-bit integers there. When converted to
decimal, 0x47 and 0x42 were 71 and 66, respectively. I opened the mobile application to see if I missed any interesting statistic, and
I actually found two additional ones that the web application was missing - elevation gain and elevation loss. Elevation gain for this
run was 71m, and elevation loss was 66m. Neat!

That left me with only two unknown bytes, but they didn't seem interesting, nor relevant at the moment, so I decided to ignore them completely.

## Exporting heart rate data

One thing that was still missing was the heart rate data. I'm not using a heart rate monitor, so I searched the internet for some gpx
file that had heart rate data included, because I was lazy to read the gpx file format specification. I quickly found this file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gpx>
  <metadata>
    <time>2010-06-26T10:06:11.000Z</time>
  </metadata>
  <trk>
    <trkseg>
      <trkpt lon="-73.9665832519531250" lat="40.7780151367187500">
        <ele>36.186767578125</ele>
        <time>2010-06-26T10:06:11.000Z</time>
        <extensions>
          <gpxtpx:TrackPointExtension>
            <gpxtpx:hr>148</gpxtpx:hr>
          </gpxtpx:TrackPointExtension>
        </extensions>
      </trkpt>
      <trkpt lon="-73.9665756225585938" lat="40.7780151367187500">
        <ele>35.2254638671875</ele>
        <time>2010-06-26T10:06:12.000Z</time>
        <extensions>
          <gpxtpx:TrackPointExtension>
            <gpxtpx:hr>152</gpxtpx:hr>
          </gpxtpx:TrackPointExtension>
        </extensions>
      </trkpt>
      <trkpt lon="-73.9665756225585938" lat="40.7780151367187500">
        <ele>34.26416015625</ele>
        <time>2010-06-26T10:06:17.000Z</time>
        <extensions>
          <gpxtpx:TrackPointExtension>
            <gpxtpx:hr>147</gpxtpx:hr>
          </gpxtpx:TrackPointExtension>
        </extensions>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

I uploaded the file to my account and again followed the network traffic from the Runtastic mobile application to see how the heart
rate data was retrieved. I quickly noticed that this time response for the activity list request contained additional
**heartRateAvailable** attribute set to true. When application receives that information, it sends additional HTTP request to the
server with the following request body:

```json
{
  "includeHeartRateZones": "true",
  "includeHeartRateTrace": {
    "include": "true",
    "version": "1"
  }
}
```

The server responds with heart rate data in a manner very similar to GPS trace response:

```json
{
  "runSessions": {
    "heartRateData": {  
      "avg": "150",
      "max": "152",
      "trace": "AAAAAwAAASlzuLI4lAAAAAAAAAAAAAAAASlzuLYgmAAAAAPoAAAAAQAAASlzuMmokwAAABdwAAAA\nAQ==",
      "version": "1",
      "count": "3"
    },
  }
}
```

It looked like the heart rate data was encoded in the same way GPS data was encoded, so I tried to decode it using the same procedure,
and I got the following result:

```
00 00 00 03
00 00 01 29 73 b8 b2 38 94 00 00 00 00 00 00 00 00 00
00 00 01 29 73 b8 b6 20 98 00 00 00 03 e8 00 00 00 01
00 00 01 29 73 b8 c9 a8 93 00 00 00 17 70 00 00 00 01
```

Following the same procedure I used for decoding the GPS trace, I found values 148, 152 and 147 at the position 8. The time info was still
at the first 8 bytes. The remaining 8 bytes looked familiar, so I downloaded the GPS trace for the activity, which looked like this:

```
00 00 00 03
00 00 01 29 73 b8 b2 38 c2 93 ee e4 42 23 1c b0 42 10 bf 40 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 01 29 73 b8 b6 20 c2 93 ee e3 42 23 1c b0 42 0c e6 e0 00 00 40 14 02 d9 00 00 03 e8 00 00 00 01 00 00 00 01
00 00 01 29 73 b8 c9 a8 c2 93 ee e3 42 23 1c b0 42 09 0e 80 00 00 00 00 00 00 00 00 17 70 00 00 00 01 00 00 00 02
```

The last 8 bytes of the heart rate data were the same as the bytes 26-33 in the GPS trace (total time and total distance).
This meant that the only useful info in the heart rate data was the single byte with the actual heart rate, because time,
total time and total distance could already be found in GPS trace. With this knowledge, I had every information I needed to
manually export my activities.

## Final words

[Runtastic Archiver](https://github.com/Metalnem/runtastic) was updated to use the GPS trace and heart rate data from the Runtastic API.
The API was extracted into separate Go package, so if anybody wants to use it, it can be imported using `github.com/metalnem/runtastic/api`
import path. Hopefully, this way of exporting activities should work for the foreseeable future, because this time Runtastic can't
actually do much, short of preventing it's users from using the application.
