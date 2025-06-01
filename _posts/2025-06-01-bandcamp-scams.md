---
layout: post
title: "How scammers trick fans on Bandcamp—and how to protect yourself"
image: /assets/img/2025-06-01-discover.png
date: 2025-06-01 17:50:00 +0200
---
It's Friday, and *Invincible Shield*, the long-awaited Judas Priest album, is finally out. To your
surprise, it's available on Bandcamp! You buy the album immediately, ecstatic that your favorite band
has finally started releasing their music on your favorite music platform. You wake up the next day,
only to find the album has vanished from your collection—you've fallen victim to a scam.

Fake artist pages often appear on Bandcamp, especially on major album release days. Anytime you see a new
page for a big name (such as Iron Maiden, Taylor Swift, John Coltrane, Pantera, James Brown, or Apocalyptica),
there's a 99.99% chance the page is a scam. Bandcamp does an excellent job of disabling these pages quickly.
Some slip through the cracks and survive for several days or even weeks, but most are removed within a day.
But even just a few hours is enough for buyers to lose money they will never get back.

Even though I'm an experienced user (I have purchased over 2,000 albums on Bandcamp over the past decade),
I too have been deceived by convincing fake pages. That got me wondering: how much money is lost to scams on
Bandcamp each month? In today's post, I'll show you how I calculated that number and share some tips to help
you avoid getting scammed.

## Collecting the data

If you are a Bandcamp user, you are probably familiar with their music discovery feed. The feed can show
best-selling albums, new arrivals, or just random releases. While the list of new arrivals isn't especially
useful for discovering quality music, it was a key piece in my quest to figure out how much money fans
lose to scams.

![](/assets/img/2025-06-01-discover.png)

All new releases show up in the feed. When Bandcamp detects a copyright violation or any other shenanigans,
they disable the offending album, and it disappears from the feed. However, if you happen to remember the
album's ID, you can still use the Bandcamp API to retrieve the price and the number of fans who had bought
it before it was disabled. That gave me an idea to periodically take a snapshot of all new releases, so I
could track which ones were removed.

And that's exactly what I had been doing in April 2024. Every day, I took a snapshot of the new releases
feed. After one month of collecting data, I had the IDs of all albums released in April. With these IDs,
I could check whether the albums had been disabled—and if they were, calculate how much money fans had
spent on them. After a few hours of writing code to summarize the raw data, I had the numbers.

## The statistics

Before diving into the topic of money, let me first show you some volume statistics:

- **81,391** albums were released in April 2024.
- **6,475** of those were eventually disabled.
- **5,158** disabled albums were never purchased by anyone.
- **1,317** disabled albums were purchased at least once.
- **1,287** different Bandcamp users were scammed.

The sheer number of releases significantly exceeded my expectations. But the number of disabled albums surprised
me even more—I didn't expect it to be in the thousands. Thanks to Bandcamp's quick reaction time, most fake albums
were never purchased. But 1,317 were, and in the next section, we'll focus on those. Let's see how much money was
lost on them.

## Let's talk money

April 2024 was a month of high-profile releases. Believe it or not, both Beyoncé and Taylor Swift
released new albums! It was an ideal situation for scammers, but they didn't turn it into a profit.
Fake Beyoncé's *Cowboy Carter* took in only £34, and fake Taylor's *The Tortured Poets Department*
just €14. Fake Dua Lipa was more successful: fake *Radical Optimism* tricked unsuspecting fans into
spending $227.76.

The three most successful fake albums were also new releases:

- Pearl Jam - *Dark Matter* (**$1,212**)
- Vampire Weekend - *Only God Was Above Us* (**$1,210.99**)
- Bladee - *Cold Visions* (**$552**)

Sadly, two of these crossed the $1,000 threshold, showing that a well-timed scam can be a lucrative
business. Many of these scam campaigns were cleverly designed to include entire discographies—not just the
new album—so the amount of stolen money per artist was even higher:

- Vampire Weekend (**$2,123.16**)
- Pearl Jam (**$1,408.44**)
- DJ SickMix (**$1,140**)

But enough about individual statistics—let's get to the numbers you've all been waiting for! The total
amount of money lost to scams in April 2024 was **$18,773.55**, **£5,364.49**, and **€1,670.73**. Altogether,
that adds up to approximately **$27,280** or **€25,420**. That's quite a lot of money!

The good news is that you can easily protect yourself from a scam by following just a few simple practices, which
I'll describe in the next section.

## Recommendations

Scammers on Bandcamp are clever—they often create pages that are visually indistinguishable from
the real thing. Here's a screenshot of one of many fake pages impersonating Norwegian musician Ihsahn:

![](/assets/img/2025-06-01-ihsahn.png)

Everything is there: high-resolution album cover, social media links, and even tour dates. It might seem
impossible to differentiate the fake page from the real one, but a few simple tips can go a long way.

#### Avoid sketchy subdomains

First, let's explain what a subdomain is. It's super simple: if a band's page URL on Bandcamp is
[immolation.bandcamp.com](https://immolation.bandcamp.com), their subdomain is *immolation*. On
Bandcamp, each label and artist has their own subdomain.

You should start paying attention to subdomains, and avoid them if they look sketchy. This is
easier said than done, though. Real artists often form their subdomain by adding a suffix to their
name—a city, state, genre, or just random-sounding abbreviation:

- Tower (from New York) is *towernyc*
- Ripped to Shreds (a death metal band) is *rippedtoshredsdeathmetal*
- Cryptopsy is *cryptopsyofficial*
- Deceased is *the-true-deceased*
- Black Curse is *blackcurse-svr* (*svr* stands for Sepulchral Voice Records)

This common practice gives scammers the opportunity to use an infinite number of plausible-sounding
subdomains for their fake pages.

Despite this, you can still spot most of the fake subdomains fairly easily. For example, Taylor Swift would
never release an album under the *quelquunquiexiste* subdomain, and Beyoncé would never use the *d-jam* subdomain.
Album names as subdomains may look more convincing, but they are almost never used by real artists.
This means *radicaloptimism* is not Dua Lipa's real page.

Sometimes, scammers use an authentic-looking subdomain, such as *pearljam* or *vampireweekend*. In those cases,
you'll need the techniques from the next two sections.

#### Be patient

If it looks like a major artist has just released an album for the first time ever on Bandcamp, chances are
it's a fake. The best thing you can do is wait at least a few days—or better yet, a few weeks. If the
page hasn't been taken down during that time, you can be much more confident that the album is legit.

#### Contact the label

When your favorite label shows up on Bandcamp with tons of releases, there's still no guarantee that the
page really belongs to them. Check out their Facebook or Instagram—if they don't mention their new Bandcamp
page, that should be a red flag. In some cases, their socials might even warn you about the ongoing scam,
like this label [did](https://www.facebook.com/quasipop/posts/1261359247613754). If you want to be 100% sure,
find the label's email and ask whether the new Bandcamp account really belongs to them. The response you are
hoping for looks something like this:

> Yepp, this is really us, all legit!

## Conclusion

To end on a positive note: I still think Bandcamp is by far the best music platform on the internet. The number
of fake albums is far outweighed by the fact that, in the past year alone, fans have paid real artists over $16
million per month on average. And now that you've learned how to better protect yourself, hopefully scammers
won't affect you at all!

Listen to great music, support independent artists, and stay tuned for my upcoming post. In the crowning
jewel of my investigative blogging, you'll read the wild story of the scammiest scammer of them all.

<small><i>Huge thanks to my wife, Milica Miljkov, for editing this post and reminding me that no matter
how good ChatGPT gets, she's still better. (Yes, I tried to replace her. No, it didn't work.)</i></small>
