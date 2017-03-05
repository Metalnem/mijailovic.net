---
layout: post
title: "Removing DRM from all Future plc magazines"
---
# Removing DRM from all Future plc magazines

While I was working on removing DRM from Edge magazine, the Edge application for iOS was constantly bombarding me with
ads for different magazines from the same publisher. That got my attention, because I previously noticed
that Edge magazine didn't have its own webpage (I purchased my yearly subscription using
[MyFavouriteMagazines](https://www.myfavouritemagazines.co.uk/) store).
One more thing I found interesting was that the application's REST API looked quite generic.
So, what if all other magazines from Future Publishing were using the same API and download format?
If that was the case, I could make my DRM removal application work with all of them.

My initial goal while writing the application was to be able to create the backup of the Edge magazine issues that I myself purchased and to
also have the option to read them in the application of my choice. I didn't expect that other people
would find it useful, because I assumed there weren't that many digital subscribers, especially
in the time of great free online gaming portals such as [Kotaku](http://www.kotaku.com)
and [Eurogamer](http://www.eurogamer.net/). But the list of magazines published by Future Publishing included other magazines,
such as [PC Gamer](https://www.myfavouritemagazines.co.uk/gaming/pc-gamer-magazine-subscription/#digital), [Total Guitar](https://www.myfavouritemagazines.co.uk/music/total-guitar-magazine-subscription/#digital) and [Total Film](https://www.myfavouritemagazines.co.uk/film/total-film-magazine-subscription/#digital). These magazines
sounded like they could be very popular, so an audience for my application could exist after all.
I started testing my hypothesis with their most popular magazines first.

## Initial experiments

I started measuring popularity using the most sophisticated scientific method available to the humanity - comparing
the numbers of Twitter followers. PC Gamer, Guitar World and Total Film had hundreds of thousands of
followers, so I downloaded iOS app for each of them. MyFavouriteMagazines web store
didn't have the option for buying single digital issue (only quarterly and yearly subscriptions were available).
Luckily, single issues were being offered as in-app purchases.

I purchased one issue of each of the previously mentioned magazines and tried running my
Edge magazine downloader too see if maybe it would work automagically. Nothing happened, of course. I could still
see the list of Edge magazine issues only. Does the API have in the requests something that is specific to the magazine
it's working with? Then I remembered the *appKey* and *secretKey* parameters - they were probably different for
each application. I fired up the Burp Suite and confirmed that was indeed the case.
I replaced the hard-coded keys for Edge with keys for PC Gamer that I just retrieved, but the I still couldn't see the
issue I just purchased. What was going on? I inspected the traffic from the mobile app again, but I couldn't see
anything unusual. The iOS app was making the same request that I was making in my application, but the difference
was the iOS app was actually receiving correct list of purchases. I was confused.

My only idea was to reinstall
the app, which proved useful - the list of purchases was empty this time. Then it dawned on me I was
actually never logged in with my MyFavouriteMagazines credentials during the in-app purchases,
so they had to be tied to my iTunes account instead! Indeed, this time in the library section of the app, there was a big
"Restore iTunes Purchases" button. Pressing the button triggered a network call to Apple servers, and my purchase
was restored. This was not good. Because I purchased my subscription of Edge magazine on MyFavouriteMagazines, it was tied to my account there.
But in-app purchases were stored in my Apple account only (I didn't even need MyFavouriteMagazines account).
Was there any way to make my application work with this scenario in mind? There were some good news and some bad news.
The good news was that the workaround exists. The bad news was that it's not very useful for the most computer users.

## Account UID

I mentioned in the previous post that every application initially calls **/createAnonymousUser/** API endpoint on the server.
That call returns UID which is then used in every following request. After you restore your purchases, they are attached to your
UID on the server and you will receive correct list of your purchases in every request after that, using the same API.
The good thing about UID is that it's generated only once, after the application is started for the first time.
That means if you extract the UID from your phone, you can use it anywhere to authenticate your requests forever.
I added an additional command line parameter to my application to confirm that. It was working! That was all good,
but how do you extract the UID from the phone in an user-friendly way? You could always install Burp Suite, configure
your phone to use it as a proxy, follow the application traffic and extract the UID parameter from any request's HTTP body.
The problem with that solution is that only tech-savvy users would manage to do that, because there are just to many
painful steps in the process. Unfortunately, I don't have anything better to offer.

The lesson here is that
you should always purchase Future Publishing magazines directly on the MyFavouriteMagazines store if you
care about making backups or reading your magazine in some other reader. Purchases you made on the web store
are available on all your devices (both iOS and Android), so that's another point in favor of the web store.

After making my application work with the UID parameter, I was finally able to test if it's working
with three magazines that I purchased. I was expecting that it would fail on the PDF decryption step, because I assumed
each magazine would have a different password. But that was not the case! Application run without a problem
and produced the PDF of the single issue that I purchased. That made my job much easier. The only thing
left was to find all appKey/secretKey combinations, but before that I had to find the list of all magazines
published by Future Publishing.

## Available magazines

There was no obvious way to list all available magazines on the web store. Fortunately, the page
with the instructions for downloading digital issues for the iOS contained what seemed to be the complete list.
After collecting all the magazines from that list, I went to Future Publishing developer's page on the iTunes
store to see if there are some additional magazines there. Here is the combined list:

[3D World](https://www.myfavouritemagazines.co.uk/design/3dworld-magazine-subscription/#digital)  
[APC Australia](https://itunes.apple.com/us/app/apc-australia-worlds-longest/id722803923)  
[Australian T3](https://itunes.apple.com/us/app/australian-t3-gadget-technology/id481969004)  
[Comic Heroes](https://www.myfavouritemagazines.co.uk/film/Comic-Heroes-Print.html#digital)  
[Computer Arts](https://www.myfavouritemagazines.co.uk/design/computer-arts-magazine-subscription/#digital)  
[Computer Music Magazine](https://www.myfavouritemagazines.co.uk/All-Magazines/Computer-Music-Print.html#digital)  
[Crime Scene](https://www.myfavouritemagazines.co.uk/film/crime-scene-magazine-subscription/#digital)  
[Digital Camera World](https://itunes.apple.com/us/app/digital-camera-world-slr-photography/id468374968)  
[Edge](https://www.myfavouritemagazines.co.uk/gaming/edge-magazine-subscription/#digital)  
[Future Music](https://www.myfavouritemagazines.co.uk/music/future-music-magazine-subscription/#digital)  
[GamesMaster](https://www.myfavouritemagazines.co.uk/gaming/gamesmaster-magazine-subscription/#digital)  
[GD Legacy](https://itunes.apple.com/us/app/gd-legacy/id454424893)  
[Guitar Techniques](https://www.myfavouritemagazines.co.uk/music/guitar-techniques-magazine-subscription/#digital)  
[Guitar World Magazine](https://itunes.apple.com/us/app/guitar-world-magazine/id469908707)  
[Guitarist](https://www.myfavouritemagazines.co.uk/music/guitarist-magazine-subscription/#digital)  
[ImagineFX](https://www.myfavouritemagazines.co.uk/design/imaginefx-magazine-subscription/#digital)  
[iPad User](https://www.myfavouritemagazines.co.uk/tech-gadgets/ipad-user-magazine-digital/#digital)  
[Linux Format](https://www.myfavouritemagazines.co.uk/All-Magazines/Linux-Format-Print.html#digital)  
[Mac Life](https://www.myfavouritemagazines.co.uk/Mac-Life-Print.html#digital)  
[MacFormat](https://www.myfavouritemagazines.co.uk/All-Magazines/MacFormat-Print.html#digital)  
[Maximum PC](https://www.myfavouritemagazines.co.uk/Maximum-PC-Print.html#digital)  
[Minecraft Mayhem](https://itunes.apple.com/us/app/minecraft-mayhem-independent/id1137912564)  
[N-Photo](https://itunes.apple.com/us/app/n-photo-nikon-photography/id479869761)  
[net magazine](https://www.myfavouritemagazines.co.uk/design/net-magazine-subscription/#digital)  
[Official Xbox Magazine](https://www.myfavouritemagazines.co.uk/gaming/official-xbox-magazine-subscription/#digital)  
[Paint & Draw](https://www.myfavouritemagazines.co.uk/design/paint-and-draw-magazine-subscription/#digital)  
[PC Format Legacy](https://itunes.apple.com/us/app/pc-format-legacy/id451451278)  
[PC Gamer](https://www.myfavouritemagazines.co.uk/gaming/pc-gamer-magazine-subscription/#digital)  
[Photography Week](https://www.myfavouritemagazines.co.uk/all-mags-all-variants/Photography-Week-iPad-iPhone-Edition.html#digital)  
[PhotoPlus](https://itunes.apple.com/us/app/photoplus-canon-photography/id451453783)  
[Pi User Magazine](https://itunes.apple.com/us/app/pi-user-magazine/id1156938158)  
[PlayStation Official Magazine](https://www.myfavouritemagazines.co.uk/gaming/official-playstation-magazine-subscription/#digital)  
[Practical Photoshop](https://www.myfavouritemagazines.co.uk/all-mags-all-variants/Practical-Photoshop-iPad-iPhone-Edition.html#digital)  
[Professional Photography Magazine](https://itunes.apple.com/us/app/professional-photography-magazine/id1039872746)  
[Rhythm](https://www.myfavouritemagazines.co.uk/music/rhythm-magazine-subscription/#digital)  
[SFX](https://www.myfavouritemagazines.co.uk/film/sfx-magazine-subscription/#digital)  
[T3](https://www.myfavouritemagazines.co.uk/tech-gadgets/t3-magazine-subscription/#digital)  
[TechLife Australia](https://itunes.apple.com/us/app/techlife-australia-tech-mag/id722804263)  
[Total Film](https://www.myfavouritemagazines.co.uk/film/total-film-magazine-subscription/#digital)  
[Total Guitar](https://www.myfavouritemagazines.co.uk/music/total-guitar-magazine-subscription/#digital)  
[Windows Help & Advice](https://www.myfavouritemagazines.co.uk/All-Magazines/Windows-Help-Advice-Print.html#digital)  

You can purchase the subscription for the most of the magazines using the web store. Unfortunately, there are
several magazines whose subscriptions are available only as an in-app purchases. Also,
[Official Xbox Magazine](https://itunes.apple.com/gb/app/xbox-the-official-magazine-uk/id451445772) and
[PlayStation Official Magazine](https://itunes.apple.com/gb/app/playstation-official-magazine/id451455977)
didn't have the app in the US iTunes store, so I had to create UK account to download them.

Now that I had the the list of all available magazines, I was ready to extract all appKey/secretKey combinations.
There was no way to automate this. At the very least, I had to download all the apps manually.
After one hour of downloading all the apps and one more hour of starting each of them while the phone is proxying the request
to Burp Suite on my desktop, I finally had all appKey/secretKey combinations.

All secret keys were 32 byte hexadecimal values. Maybe there were SHA256 hashes? Even though
that information was completely unnecessary, I wanted to find out if they were hard-coded or
generated in the app. I still had my jailbroken iPod Touch, and in the meantime I had
purchased full Hopper license, so I started digging through Edge magazine binary again, this time looking
for appKey/secretKey values. I took me a while, because of my extremely limited reversing knowledge, but I finally
discovered that application was pulling those values from **EFSConfig.plist** file.
Here is the configuration file for the **PlayStation Official Magazine (UK)**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>EFS_APP_KEY</key>
	<string>BWaG8ASaR6uXjr7wjM2wjQ</string>
	<key>EFS_CDN_SERVER_ADDRESS</key>
	<string>http://media.futr.efs.foliocloud.net</string>
	<key>EFS_DEBUG_MODE</key>
	<string>NO</string>
	<key>EFS_SECRET_KEY</key>
	<string>cde4fcb6f6d4dba10b3e52e8bb05ec51</string>
	<key>EFS_SERVER_ADDRESS</key>
	<string>https://api.futr.efs.foliocloud.net</string>
</dict>
</plist>
```

This file is shipped with the application, along with other application resources, so there was
no way to see how these values were generated. It was time to move on.

## Testing

There are a total of 41 magazines. I wanted to test if the API was working for all of them.
If I wanted to do that manually, if would obviously be a very slow and painful process. Fortunately,
it's very easy to automatize the process. There is one really nice feature of **testing** package
since Go 1.7 that I have to mention - [subtests](https://blog.golang.org/subtests). Subtests enable
you to run table-driven tests independently of each other, so if one test case fails, other will
still continue to run. You can also run your tests in parallel, which will result in a large
speedup in most cases. The resulting testing code looks like this:

```go
func TestNewSession(t *testing.T) {
	for _, mag := range magazines {
		t.Run(mag.name, func(t *testing.T) {
			t.Parallel()

			ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
			defer cancel()

			if _, err := NewSession(ctx, mag); err != nil {
				t.Fatal(err)
			}
		})
	}
}
```

These few lines of code enabled me to find out if all appKey/secretKey combinations
are working in less than 5 seconds. After adding the command line parameter for
selecting the magazine you want to work with, my application was complete.

## Future improvements

You can download fully functional Future plc downloader [here](https://github.com/Metalnem/future-plc-downloader).
Nevertheless, there are still a lot of improvements that I plan to implement at some point in the future.

### GUI

My application is aimed towards ordinary computer users, so having only command line version is not the greatest option.
There are no cross-platform GUI toolkits for Go that I would be happy to use. For some time I
was thinking about developing some command-line GUI using something like
[termui](https://github.com/gizak/termui) or [gocui](https://github.com/jroimartin/gocui), but
I ultimately decided that if I'm going to invest the time in creating nice user interface, it
better be the native application. I was also thinking about using [Electron](https://electron.atom.io/),
but I'm not very much interested in learning it.

One really nice thing about the API is that it provides cover images for each magazine issue, so there
is a possibility to create really nice GUI. I'm well familiar with the WPF, so my next step will probably be to
create at least a Windows version of the application.

### More testing

Each of the magazines has several free issues. I wanted to use them to test the complete process
of generating the final PDF. After some testing I discovered that the API for free
issues is slightly different, so I left that part for some other time.

### Removing media links

There is one interesting thing that is unique to the iOS application. Each downloaded magazine contains not just
the PDF files, but also a bunch of high-res images and videos. They are made available via embedded
links that look like this:

![](/assets/links.png)

After clicking the link, image or video is displayed in the app as an overlay. This is made possible
by the fact that the app can register custom media handlers and display the images
inline when they are triggered.

These links are useless in the final PDF that I'm generating, so I would very much like to remove them.
I opened one PDF in the text editor and found that those links have URLs looking like this:

[media://h_astro_digi-1.jpg](media://h_astro_digi-1.jpg)

Removing the links might be tricky, though, because PDF is extremely complex format (the spec is more
than 1300 pages long) and libraries that are working with it are very limited in functionality.

I tried removing the links using Adobe Acrobat and observing changes in the resulting file. It
was completely rewritten by the single change. It looks to me that each object in the file has
a stream number and there are no gaps between them, but I didn't have the time to investigate more.
This is obviously not very crucial thing to implement, but I would still be very nice thing to have.
