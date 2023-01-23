---
layout: post
title: "Advanced lesson in procrastination"
date: 2023-01-23 16:45:00 +0200
---
This story started while I was preparing my blog post
about [piracy]({{ site.baseurl }}{% post_url 2023-01-15-piracy %}). The idea was to make it clear
that I am all for supporting artists with my money, while arguing that in some cases using piracy
might be justified.

I hadn't even completed two sentences of the post before my brain decided to sabotage me with a
seemingly innocent suggestion: "Hey, you should totally calculate the value of your Bandcamp
collection, I bet it would make your position stronger!" Ok, mister brain, challenge accepted,
or whatever. I have 1,303 albums, average album price is $10, so my collection is worth roughly
$13,030. Can we move on now? "Oh, but that's just the *estimate*—why don't you calculate the
*exact* value?"

The challenge was too fun to resist, and that's how we ended up here. In this post, I will teach
you how to properly avoid doing the important work by turning a 10-second job into a weeklong odyssey.

## Purchases page

![](/assets/img/2023-01-23-purchases.png)

Bandcamp conveniently lists all your purchases on a single page (well, in my case only after scrolling
down 130 times). You can save that page to a file and then parse the HTML content to extract the prices.
That's exactly what I planned to do, but my brain interfered again. "Come on, are you really going to
scrape the HTML? You call yourself a master hacker, you should be ashamed of yourself!" I couldn't argue
against this impeccable logic—I was indeed ashamed and beaten into submission once more.

## Using the Bandcamp API

I fired up the developer tools and immediately found the order history API. It was super easy to
use, and I was only a few lines of code away from finding out the exact value of my collection. But before
doing that, I brilliantly decided that now was the best time to catch up with all fancy new .NET features
that I've been ignoring for years, such as
[Async Streams](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/async-streams)
and high-performance JSON processing using the
[System.Text.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/how-to?pivots=dotnet-7-0)
library. After putting it all to (questionable) use, I ran out of excuses and finally ran the code:

```
   64.45 AUD
  271.35 CAD
   31.20 CHF
   50.00 DKK
 2582.63 EUR
  475.30 GBP
16056.00 JPY
  350.65 NOK
    5.00 NZD
  641.60 PLN
  599.13 SEK
   12.10 SGD
 9004.38 USD
```

God damn it, what was I supposed to do with 13 different currencies? I was now in an even worse position
than at the beginning: even though these numbers were correct, it was impossible to put all of them in a
sentence. Can you see where this is all going? Yes, it was currency conversion time!

## Currency conversion

The first idea that came to my mind was to use the current exchange rates, but that would have been
ridiculous. Today's rates are irrelevant for purchases made many years ago—I needed *historical*
exchange rates instead. Searching for free currency exchange rate APIs reminded me how much I hate
the internet today. All I could find were generic, sponsored, contentless articles with names like
"25 best free currency APIs in 2022", where meaningless statements such as "bank-grade 256-bit SSL
encryption" easily earn you the top spot in the list. In this endless sea of adverts for websites
falsely claiming to be free, I somehow managed to stumble upon
[Exchangerate.host](https://exchangerate.host/#/), a hidden little gem that offers a free currency
conversion API, but for real. It was so refreshing to find a great, free product made with love in
the age where almost everything is just a hustle to make some money. Kudos to you, my unknown neighbor
from Slovakia! Anyway, I quickly figured out how to use this API, but as it happens in every good
procrastination story, this was not the end.

## European Central Bank

Home page of the [Exchangerate.host](https://exchangerate.host/#/) says that "currency data
delivered are sourced from financial data providers and banks, including the European Central Bank".
Wait, if they are getting the data from the European Central Bank, what's preventing me
from doing the same? Nothing, other than the fact that it would be completely unnecessary,
because I already had a free API I knew how to use. Logical thinking didn't stop me, though,
so I searched for "European Central Bank exchange rates" and found this:

[Euro foreign exchange reference rates](https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/index.en.html)

Latest reference rates, historical time series—everything was there, in PDF, CSV, and XML! I don't
remember why, but I chose to go with the XML format. Trying to figure out how to correctly parse the
unnecessarily complicated XML schema made me feel dumb for a while, but I ultimately succeeded
and correctly concluded that XML namespaces are dumb, not me (but Jesus, Nemanja, what the hell
were you thinking: why didn't you just parse the simple CSV file instead of wasting your time with
XML?). Anyway, the final ingredient was in my hands!

## snake_case

It's not enough to just come close to the finish line. You still need to cross it, and that's
sometimes surprisingly difficult. In every side quest you can find an even more useless side
quest hidden inside. Here is what caused my final detour (note that at this time I already had
all the info I needed to finish the quest I had embarked on):

```csharp
[JsonPropertyName("item_title")]
public string ItemTitle { get; set; }

[JsonPropertyName("unit_price")]
public decimal UnitPrice { get; set; }

[JsonPropertyName("payment_date")]
public string PaymentDate { get; set; }
```

These are the properties of the class I used for deserializing Bandcamp API responses. You can see
that I had to manually annotate them with names of underlying JSON fields. I was annoyed with that,
especially because I knew it was possible to automatically convert the names during deserialization,
at least for `PascalCase` and `camelCase`. But I didn't know the name of this letter case with
underscores, so I used my crazy Google skills to search for—wait for it—"case with underscores". I
learned that this case is named `snake_case`, but I also discovered that `SCREAMING_SNAKE_CASE` and
`kebab-case` are a thing, too. All of this had led me nowhere, though, because I soon discovered
that snake case is not supported in .NET 7. It was finally the time to let go, which I did, but not
before I read the whole GitHub thread about the plans to
[add snake_case support for System.Text.Json](https://github.com/dotnet/runtime/issues/782).

## Results

After seven days of pointless work, my mission was finally accomplished—I found out the exact value
of my Bandcamp collection! Remember how my initial guess was $13,030? Brace yourself before your hear
the exact value: it's $13,260. That's... pretty much the same number. Was it worth it the effort?
Absolutely not, but it was at least fun.

Leaving [Bandcamp calculator](https://github.com/Metalnem/bandcamp-calculator) behind me,
I had no other option but to focus and write the
[piracy post]({{ site.baseurl }}{% post_url 2023-01-15-piracy %}).

<small><i>Huge thanks to Milica Miljkov for editing this post.
Special thanks to Steven Pressfield for teaching me about the
[Resistance](https://www.amazon.com/War-Art-Steven-Pressfield-ebook/dp/B007A4SDCG/).</i></small>
