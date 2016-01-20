---
layout: post
title: BigData in Rust
---

## What I will consider "fair game"

In the previous post, I stated that not relying on a framework was freeing
the developper from the framework constraints. For instance, one can choose
to have such or such result in memory because it fits.

It also allows the developper to make use of very compact types instead of a
handful of generic purpoise ones.

But these are not choices you want to have to make all the time.

Basically, what I have done so far is spent days — weeks literally — on a
single SQL group-by query a data scientist would have crafted in less than
half a munutes :)

One of the things I'm trying to assess is what kind of tools would make
sense to developpe as a longer term goal. I'm thinking DSL, or a SharkSQL
clone or...

So now, as a developper, I
perform manually some optimisation. But I will try to stay in the realm
of optimisations that a semi-automatic optimiser could find.

```SQL
  SELECT SUBSTR(sourceIP, 1, X), SUM(adRevenue)
    FROM uservisits
GROUP BY SUBSTR(sourceIP, 1, X)
```

Somebody on twitter remarked that when I was contemplating storing the IP
characters on half-bytes, I could go all the way and just store the IP on
a proper way, that is a four-byte integer.

This is true. But both approach are unfair to the optimizer. You and
I know what to expect in a string sourceIP field (assuming the data is clean).
But there is no way an automatic optimizer can do the right thing here.

On the other hand, using a fixed array of bytes to store the result of a
`SUBSTR` is fair game. An optimiser could reasonably decide that.

As for the storage location decision ("this will fit in memory"), it's a bit
harder. I only knew the storage requirement because there were given in the
paper, by somebody who already had the query run. We could imagine various
strategies:

* ask the application developper or data scientist to provide hints
* start with an optimistic memory strategy, and start spilling to disk
    when the dataset in memory gets too big
* start with an optimistc memory strategy, restart with a disk memory gets
    too big

Anyway, all this is a long way down the road from here.

## Introducing Azone

So. I have actually crafted some code. It's not yet very
reusable but some part could be made a library as a first step towards
tools.

Some parts of it actually predate this work on Query 2, giving hopes that
there is room for reusability.

The code is here: https://github.com/kali/dazone . The README gives
instructions about how to run the bench.

Let's have a look at Query 2:
https://github.com/kali/dazone/blob/master/src/bin/query2_simple.rs

```rust
extern crate dazone;

use dazone::crunch::*;
use dazone::short_bytes_array::*;
use dazone::data::pod::UserVisits;

fn main() {
    let input = dazone::files::bibi_pod::<UserVisits>("5nodes", "uservisits", "text-deflate");
    let map = |visit: UserVisits| Emit::One(BYTES_8::prefix(&*visit.source_ip), visit.ad_revenue);
    let reduce = |a: &f32, b: &f32| a + b;
    let mut aggregator = ::dazone::crunch::aggregators::MultiHashMapAggregator::new(&reduce, 256);
    MapOp::new_map_reduce(map)
        .with_progress(true)
        .with_workers(16)
        .run(input, &mut aggregator);
    aggregator.converge();
    println!("### {} ###", aggregator.len());
}
```

Not too bad. Let's have a closer look.

### Input, Data and BIBIs

```rust
let input = dazone::files::bibi_pod::<UserVisits>("5nodes", "uservisits", "text-deflate");
```

This works, because we have:

* a Plain Old Data structure for UserVisits,
* an inflexible file organization to find files,
* a CSV library that will be compatible with Rust decodable/encodable framework,
* a deflate library.


https://github.com/kali/dazone/blob/master/src/data/pod.rs#L10

```rust
#[derive(RustcDecodable,RustcEncodable,Debug)]
pub struct UserVisits {
    pub source_ip: String,
    pub dest_url: String,
    pub visit_date: String,
    pub ad_revenue: f32,
    pub user_agent: String,
    pub country_code: String,
    pub language_code: String,
    pub search_word: String,
    pub duration: u32,
}
```

`bibi_pod` is actually quite generic. It knows the file structure
convention, so using another "table" or dataset size (the "5nodes" bit) does
not require any change. The actual pod type, `UserVisits`, is a type
parameter, so we are generic on that too. The format combination (encoding
and compression) are parsed, so `bibi_pod` knows how to read from various
compression schemes and encoding format. It also know where to read the
files accordingly.

As for its return value, it's actually:
```rust
Box<Iterator<Item=Box<Iterator<Item=UserVisits>>>>
```

That's a Boxed Iterator on Boxed Iterator of UserVisits... `bibi` for short :)

We use two levels of iterations. The top level splits data on big chunks (each
file a chunk) and the lower level is the record. It is actually quite
convenient to keep these two levels around. We load the data file by file, and
we process it by chunk anyway.

As for the Box bit, let's just ignore that for now.

### Map/Reduce

Well, map/reduce stays a relevant way to consider these kind of issues. This is
actually a bit of code that predates Query 2 and that I was able to reuse
without huge modifications. What we are looking at here is a simple, in-process
map/reduce implementation. It does not support migrating work to other
processes or nodes. We will come to that, but we will be using timely dataflow.

```rust
let map = |visit: UserVisits| Emit::One(BYTES_8::prefix(&*visit.source_ip), visit.ad_revenue);
let reduce = |a: &f32, b: &f32| a + b;
```

We will map each input file to extract the 8 bytes prefixes and the associated
revenue. The prefix will serve as a key, and the reduce stage will sum up the
revenue.

`Emit` is the return value of our map function. In my initial implementation,
the map function
was returning an Iterator, to be able to return any number of pairs. But in
the Query2 case, each record generates one and only one pair. Dealing with an
Iterator in that
case was a bit expensive, so I introduced a Zero or One or Many enumeration.
This is a terribly premature optimisation in our present case, but it predates
work on Query2.

### Running it

Now let's plug everything together and run it.

```rust
let mut aggregator = ::dazone::crunch::aggregators::MultiHashMapAggregator::new(&reduce, 256);
MapOp::new_map_reduce(map)
    .with_progress(true)
    .with_workers(16)
    .run(input, &mut aggregator);
aggregator.converge();
println!("### {} ###", aggregator.len());
```

The aggregator is a bit more complex. Here again, there are simpler,
more wasteful implementations of the aggregation. Basically, the idea is
to have part of the aggregation happen just after the map in the same
thread, construct partially reduced intermediary results and then merge them
with the "big" map instead of struggling for a global Mutex on the global
HashMap holding the result at each record.

This aggregator is called MultiHashMap because its global Map is actually
sharded to help with concurrency accesses (a bit like the ConcurrentHashMap
from the JVM).

MapOp will shove the data in the mapper and push the result to the
aggregator. We have a chance to configure it a bit, showing the progress bar
and using 16 workers.

The `converge()` call can be disregarded. Let's just print the number of
groups to the console.

All in all, this is not too bad. The fact I was able to reuse an existing
map/reduce is a good sign this that part at least could be library material.

## What's next

Next time, we will have a look at import formats.
