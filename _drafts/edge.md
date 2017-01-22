---
layout: post
title: "Removing Edge Magazine DRM"
---
# Removing Edge Magazine DRM

[Edge Magazine](https://www.myfavouritemagazines.co.uk/gaming/edge-magazine-subscription/) is one of the oldest
gaming magazines in the world. It is famous for the quality of its reviews, all of them available online up until
a few years ago, when their website was closed. The magazine continued its life in print and digital subscription
format only. I really liked the content they produced, so I decided to purchase a one year subscription for the
digital version.

Every time I buy some digital content online, the first thing I do is to make its backup and make
sure that I can use it without the official client application (if it exists). Most of the
technical literature I buy now comes in PDF format without DRM (InformIT, O'Reilly, No Starch Press). Amazon
Kindle books come with DRM that is easy to remove, and the same thing applies to Audible books. For the
Dark Horse Comics I wrote a little [tool](https://github.com/Metalnem/dark-horse-downloader) that converts
their proprietary format into CBZ, which is a format that every comic book reader supports. Digital
version of the Edge Magazine came with its own app, so there was no way to backup the magazines.
Application didn't have any interactivity, so I assumed that it was just a viewer for PDF, EPUB or some
other book format. If that was true, there must have been some way to extract the content from it.
First step towards that goal was to reverse the application's API.

## Reversing the API

I've written about reversing private APIs in the post
[Reversing Runtastic API]({{ site.baseurl }}{% post_url 2016-10-20-reversing-runtastic-api %}), so I won't get
into the details here. I'll just give you a tour of the API, with the annotated requests and responses.
API server is located at <https://api.futr.efs.foliocloud.net>. All requests are POST requests,
with the URL encoded parameters. Responses are JSON formatted.

The exchange starts by client sending the request to the **/createAnonymousUser/** endpoint.
Request contains hard-coded *appKey* and *secretKey* parameters.

```
appKey: RymlyxWkRBKjDKsG3TpLAQ
platform: iphone-retina
secretKey: b9dd34da8c269e44879ea1be2a0f9f7c
```

Server responds with the random ID that the client must include in all following requests.

```json
{  
  "data": {  
    "uid": "05681280014840428595874b26b8ab4f9991495580"
  }
}
```

Client sends its email address and password in the **/login/** request.

```
api_params: {"identifier":"imateapot@mailinator.com","password":"imateapot"}
uid: 05681280014840428595874b26b8ab4f9991495580
```

Server responds with the download ticket number. That's strange response for a login request.

```json
{  
  "data": {  
    "download_ticket_no": "00649540014842485615877d5f10fdbf6313623950"
  }
}
```

This is the moment where things get interesting. After the login request is completed and
the client received a download ticket, login process is only halfway done. Next step
is to call the **/getDownloadUrl/** endpoint with the download ticket.

```
ticket: 00649540014842485615877d5f10fdbf6313623950
uid: 05681280014840428595874b26b8ab4f9991495580
```

Server will respond with *status* field initially set to -1. Client sends the getDownloadUrl
repeatedly after that until the server responds with *status* set to 1. I never discovered
how it finally decides that it should let you in.

```json
{  
  "data": {  
    "uid": "05681280014840428595874b26b8ab4f9991495580",
    "status": 1
  }
}
```

Congratulations, you are logged in! Your status is now attached on the server to the ID
you received in the initial request. Next step is to download list of all products that
you purchased. That is done via **/getPurchasedProductList/** endpoint.

```
uid: 05681280014840428595874b26b8ab4f9991495580
```

Server responds with the list of all available issues with bunch of metadata. Only one
field is relevant to us, and that is SKU. It contains the unique name of a magazine issue.

```json
{  
  "data": {  
    "purchased_product_list": [  
      {  
        "sku": "com.futurenet.edgemagazine.300"
      }
    ]
  }
}
```

Finally, after we have the list of all purchased issues, we can ask for the download URL
of a single issue by calling the **/getEntitledProduct/** with the SKU parameter.

```
sku: com.futurenet.edgemagazine.300
uid: 05681280014840428595874b26b8ab4f9991495580
```

Server will respond with the product title and download URL.

```json
{  
  "data": {  
    "sku": "com.futurenet.edgemagazine.300",
    "product_title": "Christmas 2016",
    "secure_download_url": "http:\/\/cdn.futr.efs.foliocloud.net\/com.futurenet.edgemagazine.300.zip"
  }
}
```

That's it! When you have the download link, you can download the file without any authentication, so
I didn't include its real value in the example response.

After downloading the file for one of the issues that I had purchased, I discovered that it consists of
bunch of PDF files named *page-0.pdf*, *page-1.pdf*, etc. That confirmed my theory from the beginning.
I was happy because all that was left was to somehow merge the files. Unfortunately, when I tried to open
one of the PDFs, I was asked to provide the password. That came unexpected!

The good thing was that archive files were obviously statically hosted on the server (no user
dependent data in the download URL), so there was only one password involved
(in the more secure scheme, they could have generated the PDFs on the fly
with the different password every time). All of that meant that the password was
hard-coded in the application code in some way (either directly or derived from some other
hard-coded values using some algorithm). My next step was to extract the application
binary from my phone and somehow retrieve the password from it. But this turned out
to be harder than expected.

## iOS jailbreaking

Edge Magazine app was available only on iOS and Android,
so decompiling .NET like I did in the case of Runtastic app was out. What was even worse, there was
no standalone app on Android. Edge Magazine is downloaded via Google Play Newsstand, so I couldn't
just download the APK file and decompile it. That left native iOS application as my only choice. To extract the
binary from the phone, I had to jailbreak it.

My iPhone had the latest available iOS version, for which the jailbreak was not available. Even if was
available, I would never jailbreak my primary cell phone, so I had to find some other solution. I had
an old iPad with the iOS 9.3.5 installed on it, but I soon discovered that all available jailbreak methods
supported only iOS up to 9.3.3. Versions 9.3.4 and 9.3.5 were actually minor updates specifically
designed to prevent known jailbreak methods. The only solution left was to buy some cheap old iPhone, preferably
already jailbroken. While I was looking through online ads, I suddenly remembered that my girlfriend
had an iPod Touch that she stopped using some time ago, so it was available for me to play with.
It was iPod Touch 4th Generation, with the ancient iOS 6.1.6. Remember this?

![](/assets/img/ios.png)

Edge Magazine app required at least iOS 7, but I decided to try to install it anyway.
Somehow it had still been displayed in the app store. When I tried to download it, I was informed that
it didn't support my iOS version, but I was offered to install older version of the app instead. Good enough for me!

Now that I had the application installed, I could proceed with the jailbreak process. It was a very painful
experience. There are two available tools for jailbreaking iOS 6.1.6, p0sixspwn and Redsn0w. I found
a tutorial on [iPhone Hacks](http://www.iphonehacks.com/2014/03/jailbreak-ios-6-1-6-redsn0w-p0sixspwn.html)
webpage, but both tools failed spectacularly on my Mac, one by requiring some really old version of iTunes,
and other by giving me some unknown errors. I tried them both on Windows 7 virtual machine, where I already
had some old iTunes installed, but that also failed. This time one of the jailbreak tools just crashed every time, and
the other continued to give me errors. I finally managed to jailbreak my iPod Touch using some other laptop with Windows 10 installed.

## Extracting the binary

Jailbroken device was only one of the many steps in the process of extracting the binary.
Next thing I had to do was to enable remote access to the device by installing
[OpenSSH](https://cydia.saurik.com/package/openssh/). If you want to access terminal directly
on the device, you can install [MobileTerminal](https://cydia.saurik.com/package/mobileterminal/) too.

With all the prerequisites installed, I started looking for the binary location.
I was not familiar with the iOS file system hierarchy,
so I just did a grep search for *edge*. That gave me multiple results in the directory
`/private/etc/var/mobile/Applications/A043BAFC-8C87-40B1-BA0E-A8A673F377F0`. That looked like the
root directory of the app. I could have just started exploring directory from the terminal, but I prefer
doing that in graphical interface. While I was still using Windows, I liked [WinSCP](https://winscp.net/eng/index.php),
but on the Mac OS X I still haven't tried any alternative. Google search quickly found [Cyberduck](https://cyberduck.io/),
which looked very nice. I connected to the device with the default root credentials (`alpine` is the default root password
on every iPhone) and navigated to the folder I found.

![](/assets/img/file-system.png)

I assumed the application was in the folio.app directory (it had the app extension, after all).
The folder had a bunch of files and folders that looked like some kind of application resources.
Among them was one file called folio, with the size of about 15MB. That was probably the binary!

But you can't just copy the binary and disassemble it. All the applications on the iOS are encrypted,
so I had to find a way to decrypt it. There are several available decryption tools and I randomly
chose to use [Clutch](https://github.com/KJCracks/Clutch). Current version supported only iOS 8 or newer.
Luckily, the app was hosted on GitHub and all of the previous releases were still there. I found the one
that supported iOS 6.1.6 and copied it to the device. It was super simple to use, so I had the binary
decrypted and copied to my machine in no time. I was ready for the disassembling.

## Disassembling the binary

This was my first attempt at doing static analysis of a native application. I had experience in reversing
managed applications, but that was a much easier task. On top of that, iOS runs on ARM architecture,
and I was not familiar with ARM assembly at all.

[IDA](https://www.hex-rays.com/products/ida/index.shtml) is without a doubt the best disassembler
currently available, but it's aimed at professionals, and that shows in its price. Much more
affordable solutions for the hobbyist hackers are [Hopper](https://www.hopperapp.com/)
and [Binary Ninja](https://binary.ninja/). I've heard a lot of good things about both,
but the prevailing factor in my decision to choose Hopper was that it had the support
for retrieving Objective-C information from the analyzed files, which was useful for
a novice in the field like me. I downloaded Hopper and opened the application binary in it.
First screen that I encountered after that looked very intimidating.

![](/assets/img/hopper.png)

Fortunately, Hopper's user interface was relatively intuitive, even though it contained
a lot of information. On the left side you can see the list of all functions and strings
in the analyzed program. That was a good starting point for me. Middle part of the screen
contained the most important part of the analyzed program: assembly code with the decompiled
Objective-C code in the comments. On the right side there were a bunch of options, but the most interesting
ones were *Is Referenced By* and *Has Reference To*, which looked very useful for navigation between functions.

My first step was to try to find which part of the code was doing the PDF decryption.
I assumed that the function doing that would contain PDF in its name, so I did a search
for *PDF* in the Proc. pane. There were hundreds of functions doing PDF operations. Instead
of looking through that large list, I tried to narrow down the search by searching for *password*.
That helped a lot, because the results could fit on the screen this time. Skimming through the
list I quickly noticed the function CGPDFdocumentUnlockWithPassword. The name said everything.

I was not interested in the function itself, but rather in the parameters it received.
One of them probably had to be the password. *Is Referenced By* section proved helpful
by showing that the function was called in two places, located at
addresses 0x3dc3b6 and 0x3dc428. I navigated to the first one. That call to
CGPDFdocumentUnlockWithPassword was located inside one huge function. I started going
through it line by line. First thing I noticed was that the function was using bunch
of weird looking strings in blocks of instructions looking like this:

```
003dc01e    movw    r6, #0x2ddc    ; @"\\\"F0rdbCCI", :lower16:(0x5cee08 - 0x3dc02c)
003dc024    movt    r6, #0x1f      ; @"\\\"F0rdbCCI", :upper16:(0x5cee08 - 0x3dc02c)

003dc0ac    movw    r6, #0x2d60    ; @"vo*t3hGw", :lower16:(0x5cee18 - 0x3dc0b8)
003dc0b0    movt    r6, #0x1f      ; @"vo*t3hGw", :upper16:(0x5cee18 - 0x3dc0b8)

003dc116    movw    r0, #0x2d04    ; @"KUC%3p1c", :lower16:(0x5cee28 - 0x3dc124)
003dc11a    movt    r0, #0x1f      ; @"KUC%3p1c", :upper16:(0x5cee28 - 0x3dc124)

003dc14e    movw    r0, #0x2cdc    ; @"SjPJ&h0nkY!\\\"", :lower16:(0x5cee38 - 0x3dc15c)
003dc152    movt    r0, #0x1f      ; @"SjPJ&h0nkY!\\\"", :upper16:(0x5cee38 - 0x3dc15c)
```

These strings looked like they were being combined into a password. First started with the
escaped quote, and the last one ended with the exclamation mark and escaped quote. I tried
concatenating them in various ways in attempt to form a valid password for the PDF files,
but had no luck in doing that. At this point I knew that finding the password was just
a matter of time. I could have invested some time in learning the basics of ARM v7 assembly,
but I wanted to try few more things that, because I felt that I was very close to a solution.

I opened the Strings pane and navigated to the place where the four strings of interest
were located, hoping that I would see something interesting in their vicinity.

```
004dcff9    db    "\"F0rdbCCI", 0        ; DATA XREF=cfstring__F0rdbCCI
004dd003    db    "vo*t3hGw", 0          ; DATA XREF=cfstring_vo_t3hGw
004dd00c    db    "KUC%3p1c", 0          ; DATA XREF=cfstring_KUC_3p1c
004dd015    db    "SjPJ&h0nkY!\"", 0     ; DATA XREF=cfstring_SjPJ_h0nkY__
004dd022    db    "SavedPasswords", 0    ; DATA XREF=cfstring_SavedPasswords
```

There was a string *SavedPasswords* right next to them. Could it be that the password
is being saved somewhere? I connected via SSH to the device again, this time searching
for files containing the string *SavedPassword*. I had one hit! The file was called
`com.futurenet.edgemagazine.plist`. If looked promising, because plist is the default
configuration file format in Apple software system. I copied it to my Mac using Cyberduck,
because Mac OS has a nice viewer for plist files. Looking through the contents of the file
I found this:

![](/assets/img/password.png)

The password was the concatenation of the strings that I found before, with some of the
characters removed: **"F0rd** bCCIvo **\*t3h** GwKUC **%3p1c** SjPJ **&h0nkY!"**
I was overjoyed! It goes without saying that the password worked on downloaded PDFs.

## Unlocking and combining the individual PDF pages

My job was now still just partly done. What I had at the moment were individual PDF pages and the
password that could decrypt them, but I still had to write the application that would produce
single PDF that combined the decrypted pages. I started looking for PDF libraries written in Go. After some time, I
stumbled upon [UniDoc](http://unidoc.io/). Among the features listed on the features page there were
*Merge PDF* and *Unlock PDF*. That was exactly what I needed! What was even better is that the authors
of the library had separate GitHub repository with [examples](https://github.com/unidoc/unidoc-examples)
that included both merging and unlocking PDFs. With that knowledge, it was just a matter of hours before I had fully
functional application for downloading Edge Magazine issues.
You can download it [here](https://github.com/Metalnem/edge-magazine-downloader).
