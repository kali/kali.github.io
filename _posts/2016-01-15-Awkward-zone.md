---
layout: post
title: Awkward Zone
---

<img src="/assets/tool-1076326_1920.jpg" alt"four wrenches" height="400px">

## Rust, BigData and my laptop

I have been toying and working with BigData tools for years. Data preparation,
index building, logs processing, Wikipedia graph analysis...
Not necessarily huge
datasets, but often in the "awkward zone", where a scripting language show its
limits, but firing up a 20-node cluster does not feel right.

Some things have changed since the early 2000s. Getting access to hundreds or
thousands of computer for a few hours at an affordable rate was
science-fiction. — Have you read _Permutation City_, by Greg Egan ? — 
It's now a commonplace occurence. We have also access to
a variety of software tools to distribute computation on these clusters.

So distributed processing is now both reasonably affordable and easy, providing
a very acceptable answer to most problems that would have landed in the
"awkward zone" a few years ago.

An unfortunate consequence of making scalability accessible is... we may have,
as an industry, become lazy. If we have a scalable solution to a problem, it is
tempting to choose the easy way, just throw more hardware at the problem
without trying too hard getting a more efficient solution.

It's a bit of a shame.

The latest figures I could find about Internet energy impact put it somewhere
between 3% and 10% of the global energy balance. And that's heat. Just heat.
From a thermodynamic point of view, computers are heaters.

Also... well I think it's fun. And constraints generates creativity.

So I'll spend some time trying to explore (again) that good old "awkward zone"
that EC2 and Spark have more or less anhilated.

Let's try and do some BigData on a laptop.

## Game changers

Let's consider our post-EC2, post-Spark world. I think two items bring
something new to the table.

### Affordable SSD

We have been stuck with the 5-to-15-millisecond latency of spin-disk for years.
Charting memory speed and availibility was showing a huge empty zone between
RAM and disks. This gap is now closing with SSD — and a few other more
exotic contraptions. This is a huge thing. Most of the reasoning for data
processing in the previous decade was structured by this strict dichotomy
between fast memory and cheap memory. Everything big
was I/O bound anyway. Purely sequential access was the only way to go. CPU were
usually starved, so expensive compression was often a good option.

We have to reconsider the choices and compromises we did in the previous decade
in the light of the SSD characteristics. It will take years. And SSD is just
the beginning: now that the spin-disk barrier is crossed, we will see
various storage devices with performance characteristics and costs all over
the place.

### Rust

SSDs are here, and they are here to stay. The second game changer is a software
one, and it may just be wishful thinking on my part.

Rust is a new language, aiming at being a modern and viable alternative to
C and C++ in the "system programming language" niche. Mozilla.org, one of the
driving forces behind Rust, is bored with C++ and wants a new language to
write a new Mozilla.

As a language, Rust share characteristics with Scala and Swift, featuring a
strong trait-based type system, integration of functionnal idioms in an overall
imperative and object language. But Rust has a unique approach to resource
management: the implicit ownership and borrowing of resources that we have
always worked with have been made explicit in the language. So basically, the
Rust developper can write code that will be as efficient as C++ code, as
safe as Java, in a language supporting high-level idioms.

Yes, Rust wants it all. The price is a steep learning curve. Progresses
are being
made to help (error message improvements, smarter borrow-checker) but
making friend with the borrow-checker still dominates the Rust beginner
experience. And having a difficult time negotiating with the borrow-checker
stays a regular occurence when trying to write abstract library code.

## The BigData benchmark / Query 2

So to exercice Rust and SSD in the awkward zone, I looked around for
examples with documented performance and I found
[just that](https://amplab.cs.berkeley.edu/benchmark/).

It focuses on comparing different SQL-like batch processing engines on a
5-nodes EC2 cluster: spark SQL, hive, etc. Redshift, AWS proprietary
engine is also included in the bench.

Four queries are included in the bench:

 - simple scan and filter
 - group-by
 - join
 - external user-defined function

I will focus on the "group-by" query, called... Query 2. 

The input is a 30GB table, called "UserVisits", representing anonymised
visits on a web site. The query uses two fields (sourceIP and adRevenue) among
a dozen. The query groups visits by a prefix of the sourceIP, and sum
adRevenue on these groups. There are three variants for the query with a
8, 10 or 12 bytes prefix length (X). Note that the variant only impacts the
size of the result, not the input.

```SQL
  SELECT SUBSTR(sourceIP, 1, X), SUM(adRevenue)
    FROM uservisits
GROUP BY SUBSTR(sourceIP, 1, X)
```

Full results with graphs are provided in the benchmark page, but let's focus
on this:

| Query   |  X | group count | Hive perf | Shark perf |
|:-------:|---:|------------:|----------:|-----------:|
|   2A    |   8|   2,067,313 |      730s |        83s |
|   2B    |  10|  31,348,913 |      764s |       100s |
|   2C    |  12| 253,890,330 |      730s |       132s |

This is running on a cluster made of five 8cores/64GB nodes on EC2.

I have not shown the best-performing solution in the above table. The reason
is, as the benchmark page explains, RedShift uses a columned input. As the
query we are working on use about 15% of the actual data, we can expect a big
speed improvement there. We may consider using a columned format ourselves,
but not in this initial test.

The raw input is provided in 2037 "deflated" CSV files.

## First iteration

Now one of the interesting things about doing something like that by hand
is, there is no framework to dictate how to architect or organize the
computation. You're free.

The result is a table containing 2M or 254M records, each record being
a pair (prefix, amount). Our worst case will be a 12bytes prefix, amount a
32-bits float. So each record is 16bytes. Our theorical result size is
about 4GB. That's fine, my laptop has 16GB. Note that I chose not to write
the result to Disk in which I differ from the bigdata benchmark.

Rust structures are lean. Rust HashMap will have some overhead, but nothing
unreasonable. For the prefixes, I can use [u8;12] fixed arrays. That's just
an array of 12 (unsigned) bytes. No hidden
cost. Another option would be an actual String or Vec
(a resizable vector) which could be slightly easier to manipulate but they
would incur some overhead: Vec is a structure with a pointer to the actual
buffer, plus two integer fields for buffer capacity and actual size.
String is more or less the same. Three
words, 24 bytes. Bigger than the usefull data itself. Let's not go there.

Actually, I could be more aggressive on the keys. They are ipv4 adresses
prefix,
so each byte can only be a figure or a dot... this should take half a byte,
not one byte. Let's keep that for later

As my laptop has 4 hyperthreaded cores, I need to parallelize the computation
somehow. I picked a work stealing queue, enqueued a job for each input file.
Each job scans a file and performs a local aggregation on its own hashmap.
Once it's done it drains its own little HashMap in a big shared HashMap.

And that's more or less it.

cargo build, run, wait, look at the progress bar for a while.

Mmmm. Frown.

Ho!

cargo build --release, run, wait, look at the progress bar.

Smile.

Drum roll... 633s! We are already doing better than a 5-nodes hive cluster.
ON A LAPTOP, playing David Bowie songs to cover its fans noise,
plugged to a 4K display, running Chrome with a few dozen tabs,
and about as many iTerm panes. So not particularly quiet.

That was for the A variant. The C variant runs in 666 (!) seconds.

## Teaser

As I have hinted several times, I plan to detail more iterations in the
coming weeks.
I will show [some code]({% post_url 2016-01-25-The-Rust-is-in-there%})
(once it's cleaned), and do more stuff:
[play with various input formats and optimize]({% post_url 2016-02-01-Lets-optimize %}),
distribute the computation using
[timely dataflow](https://github.com/frankmcsherry/timely-dataflow)
on a cluster. This part is done, I just need to write about it :)

And we'll get way better than these 633s.

{% include {{site.baseurl}}/BigDataSeries.md %}
