---
layout: post
title: "How to burn the most money with a single click in Azure"
date: 2020-03-28 14:20:00 +0200
---
Corey Quinn twitted [this](https://twitter.com/QuinnyPig/status/1243316557993795586)
yesterday: ![](/assets/img/2020-03-28-corey-quinn.png)

The twit was followed by a very productive discussion,
with many nice people trying to figure out even better
ways to spend money on AWS. As an Azure user, I felt sad
and left out, so I decided to fix this injustice. If you
are in quarantine and you have insane amounts of money to
spend, I got you covered: you only need one Pay-As-You-Go
Azure subscription, and I promise I'll help you make all
your hard-earned money disappear in a second!

## Warming up

The first thing any sane person would try to do when
attempting to burn money using any cloud provider is
to buy a very large and expensive virtual machine.
Sadly, this will get you nowhere:

![](/assets/img/2020-03-28-virtual-machine.png)

One Standard_M416ms_v2 instance with 416 vCPUs
and 11,400 GB of RAM paid three years upfront will
only cost you 802K, which is not even a million!
To make things worse, Azure is giving you 71% discount,
and savings are completely opposite of what we are trying
to achieve. Credit where credit is due: Azure says that
the recommended quantity of these instances for me is zero,
which is a pretty accurate guess.

## Breaking 1M (Azure Blob Storage)

How about Azure Blob Storage? Usually, it's very cheap,
but what if for some reason you need to store one petabyte
of data in south Brazil for the next three years?

![](/assets/img/2020-03-28-azure-blob-storage.png)

Unfortunately, you can only get read-access geo-redundant
storage here, which leaves you wondering what would be the
price for read-access geo-zone-redundant storage if it was
available.

## Breaking 1M (Azure Databricks)

Our next candidate is Azure Databricks. I have no idea
what it is, and I don't even care! All I know is that
it's pretty expensive, and that's exactly what I need:

![](/assets/img/2020-03-28-azure-databricks.png)

Three million euros will buy you more than six million
of these bricks of data. At 0.53 EUR per brick, I honestly
don't know if this is a good deal or not.

## The winner

I use Cosmos DB a lot, and I know that it can get pretty
expensive, so I tried to reserve some capacity:

![](/assets/img/2020-03-28-azure-cosmosdb-20k.png)

At less than 50K EUR for 20K RU/s, this is pathetic.
When I tried to reserve 5M RU/s, Azure didn't let me
do it, because apparently I'm not "enterprise" enough:

![](/assets/img/2020-03-28-azure-cosmosdb-5m.png)

Luckily, we can at least get an estimate by using
[Azure pricing calculator](https://azure.microsoft.com/en-us/pricing/calculator/?service=cosmos-db):

![](/assets/img/2020-03-28-pricing-calculator.png)

At more then 9M USD, we have a winner! Or do we?

## Cheating

Some of you dear readers are now probably asking: why
would you limit yourself to a single petabyte of data?
That's a completely legitimate question! Let's take
all petabytes available in south Brazil:

![](/assets/img/2020-03-28-azure-blob-storage-99x.png)

It's a shame they only have 99 of them petabytes, so let's
try to reserve 999 Standard_M416ms_v2 instances:

![](/assets/img/2020-03-28-virtual-machine-999x.png)

Not bad! However, I don't have enough money on my card to
test if this operation will succeed. If you do, please click
"Review + buy" and tell me what happened.

## Conclusion

Please do this at home, and tell me your experience!
Just don't be sad if some of these ways
[don't work](https://twitter.com/virtualmanc/status/1242370080236863489)
for you right now:

![](/assets/img/2020-03-28-neil-mcloughlin.png)
