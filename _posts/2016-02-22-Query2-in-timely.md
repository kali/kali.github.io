---
layout: post
title: Query2 in timely dataflow
---

[Last week]({%post_url 2016-02-15-Embrace-the-glow-cloud%}/), we have 
established that [timely-dataflow](http://github.com/frankmcsherry/timely-dataflow) 
rocks. We have shown it
was allowing us to crunch data with one order of magnitude cost-efficiency
that redshift or Spark on EC2.

Timely is great, but it can be a bit intimidating. It's lower-level than
Spark, bringing us a bit to the Hadoop manual map/reduce era. So this week, we
will take the time to translate step by step our good old Query 2 friend to its
timely-dataflow implementation.

![rusted gears]({{ site.baseurl }}/assets/gear-1127518_640.png)

## From pseudo-SQL to execution plan

First, we need to do the query planner job by hand. Some people like SQL syntax
— you guessed it, I'm not one of them — but translating it to something
more computer friendly is not simple.

```SQL
  SELECT SUBSTR(sourceIP, 1, 8), SUM(adRevenue)
    FROM uservisits
GROUP BY SUBSTR(sourceIP, 1, 8)
```

What's not to like? Eye-hurting capitalization convention, moronic SUBSTR
repetition, complete lack of composability, absurd reading order.

Big, big sigh.

Anyway. What would it look like in Spark, or in fluent collection style? (This
is pseudo code)

```scala
UserVisits.load_from(...)
    .map(visit => (visit.sourceIP.substring(0,8),visit.adRevenue))
    .reduceByKey((a,b) => a+b)
    .count
```

This tries less harder to be readable in natural language, but is a good step
is a very nice step in the direction of actually doing something.

To paraphrase it:

* we start with a big collection of UserVisits
* for each of them, we make a Key/Value pair of an 8 position prefix of the
    sourceIp and the visit revenue
* for each Key prefix, we sum the associated revenues Values
* and in the end, we will only be checking the number of results

In all the experimentations we have done so far, I have computed the actual
revenue per prefix, even if I never displayed more than the count of prefix.

Now, if we were to run in Shark, we would be more or less done at this point,
and Shark would do the rest for us. But timely dataflow, even with
differential-dataflow, does not do this extra mile for us.

## Reducers

First, we need to decide how to distribute this to several workers. In
everything that follows, a worker is some code that shares no application data
with its couterparts. From an isolation point-of-view, a worker could be a
process — they were just that in good old Hadoop. In timely, they can be
multiple threads running on different processes, running on different servers.
But the isolation invariant stands: application logic and data stay contained
in one worker. There is no shared-across-worker memory structures visible to
the application developer.

A very efficient way to
think about execution plan, obviously, to think in map and reduce terms.

* Maps are "pure" functions of an item in a stream. A map will be run on each
    item on the stream, producing one item of output at each turn, not having
    any side effect at all. Maps are the "nice guys" in map/reduce. They are
    very composable, scale naturally. They will often contain most of the
    application logic code, but are not really interesting from a structural
    point of view.
* Reduces, of course, are another story. They are operations that for instance:
    * may have state,
    * may "break" the item-to-item stream of data (because they need to
        accumulate some or all of its input before outputting something),
    * may need to know exhaustively something about some subset or all of the
        dataset to produce a correct result.
    On the happy side, there is a small zoology of reduce pattern families.

So basically, when processing data in map/reduce, you pick and configure a few
reducers, shoves most of your application logic in the mapper, and let the
framework do it's magic. Which is more or less what we have just done with
the pseudo-code pseudo-spark code above.

It has an input, a map, a first reducer 'reduceByKey', that will perform a
reduction of numbers by summing them, and a second reducer in the form of the
final count.

That is exactly the path I followed to write the timely implementation of
Query2. From SQL to stream (at least in my head) to translation to timely.

I happened to follow a different mental path for the non-timely implementation.
If you remember the previous posts, with less constraints to start with (a
blank page) I manage to produce a slower implementation than the one
constrained by timely... even if it had only one reducer step.

Distributing Reducers is more complicated than Mappers because we must deal
with their "unit of work" and make sure it falls into one single worker.

* the reduceByKey unit of work is portion of our dataset sharing the same
    key. Remember: it receive all visit sharing an sourceIP prefix, and
    produces the total revenue for them. To provide the correct result, we
    need to make sure all prefix/revenue pairs are dealt at one single place
* the count unit of work is the whole dataset.

All in all, some data will flow over the network twice, one for each reducer.
First to get to the right worker for the reduceByKey responsible for the
matching shard of the data, later each worker will contribute a single number
of keys to an arbitrary worker which will sum them.
Timely dataflow calls these worker-to-worker communications "Exchange".

## Implementing the two-reducer plan in timely

The [full code is here](https://github.com/kali/dazone/blob/master/src/bin/query2_timely.rs).

It is not the executable I used for the EC2 tests. I used `query2.rs` which
allow me to tweak and instrument more things. I copy-pasta-ed the relevant
bits to make a query2_timely something easier to explain. It has the same
performance characteristics.

It's a single `main()` function, but the first (and only) thing it does is
yield control to timely's configuration manager. Yeah, Inversion-Of-Control,
rust-style.

```rust
fn main() {
    timely::execute_from_args(std::env::args(), move |root| {
        let peers = root.peers();
        let index = root.index();

        root.scoped::<u64, _, _>(move |builder| {
            [...]
        };
    }
}
```

This allows timely to parse the arguments from the command line, and start as
many workers as necessary. Each worker is given an index, and knows the number
of its peers. We put this useful constants somewhere convenient as we will need
them in places where borrowing `root` will not be an option — rust being rust,
it is picky about that.

The real interesting stuff happen in `scoped()`. This is where we will plug
everything together.

First we have lines 23 to 33 devoted to read the input files. There is not much
in there that is timely-specific, but let's have a look at them, some
bits are actually relevant.

```rust
let files = dazone::files::files_for_format("5nodes", "uservisits", "buren-snz");
let files = files.enumerate().filter_map(move |(i, f)| {
    if i % peers == index {
        Some(f)
    } else {
        None
    }
});
let uservisits = files.flat_map(|file| {
    PartialDeserializer::new(file, Compressor::get("snz"), &[0, 3])
});
```

First we get an iterator on all the files from the right data directory. This
is just stuff I made for these Query2 experiments. The next few lines are more
relevant: each worker will load some files in the distributed system. We want
every file to be read once and only once, and we want the load spread evenly
across the workers. This is where the index and peers count we put aside before
come in handy.

Once the files are filtered, we can load the actual content. PartialDeserializer,
Compressor are part of the Query2 experiment code. The last three lines produce
a rust standard `Iterator` over (sourceIp, adRevenur) pairs.

Next comes the SUBSTR, in a form of a `map`:

```rust
let uservisits = uservisits.map(|visit: (String, f32)| {
    (Bytes8::prefix(&*visit.0), visit.1)
});
```

Here again, nothing comes from timely. Bytes8 is a structure around an array
of eight bytes I have written for Query2. As a matter of fact, Map are so easy
to deal with that I have not even bother putting this one in timely formalism.
I could have done it, but what's the point really? Iterators are so easy...

A few more lines of preliminary:

```
let stream = uservisits.to_stream(builder);

let mut hashmap = ::std::collections::HashMap::new();
let mut sum = 0usize;
```

`to_stream` actually comes from timely! It takes a standard rust Iterator and
transforms it into a timely Stream. From a 30 000 feet point of view, think of
it as a mere adaptor.

The two other lines are standard rust, but they define the state of the worker:
the HashMap stores the on-going reduced state of the `reduceByKey` step, the
sum is the on-going figure for the final `count()`.

And now for the main course, the two reducers in all their splendor.

```rust
let group = stream.unary_notify(
    Exchange::new(move |x: &(Bytes8, f32)| {
                        ::dazone::hash(&(x.0)) as u64
    }),
    "reduceByKey/Count",
    vec![],
    move |input, output, notif| {
        input.for_each(|time, chunk| {
            notif.notify_at(time);
            for (k, v) in chunk.drain(..) {
                update_hashmap(&mut hashmap, &|&a, &b| a + b, k, v);
            }
        });
        notif.for_each(|time, _| {
            if hashmap.len() > 0 {
                output.session(&time).give(hashmap.len());
            }
        });
    });
```

First stage implements the reduceByKey — and half of the count actually.
Obviously, we are deep in timely realm this time.

We plug a unary_notify operator on our uservisits stream, and we will keep
the resulting stream in `group` for our next step.

`unary_notify` is "unary" in that it define an operator that has one input
(the uservisits key/value tuples) and one output. It is "_notify" in that,
as we are a reducer and not a map, we will need some access to the shared
computation progress in order to know when have received everything that
belong to our shard of data. Access to this global state is why "timely" is
called "timely": each record in the stream is tagged with a "time". Time
is actually a complex thing in timely dataflow. Think about the black board
scene in Back to the Future II, combined with Interstellar library and twelve
monkeys mad scientists — or mad engineers, sometimes it's hard to tell the
difference.

In our example, it's quite simple. There is just one "epoch" we are interested
in for the reduceByKey: we need to be notified when all the workers in the
system are done with the input files.

The `Exchange` is what will decide to which broker a record should be send:
we are returning a hash of the key from the pair. Timely will make sure all
records with the same hash are going to the same worker.

Next come a debug and audit name for our worker, and a vector of timestamps
we know we will want to be notified at. We could use this to register our
the required notification, except we don't know what to_stream() uses as
timestamp. Fortunately, there are other way to register notifications.

Finally, the last argument of unary_notify is the logic that the operator
will run every time it gets a chance. It takes the form of a closure which
allow the logic to make use of the input, output and notification bus, and,
by capture, the various variables the worker has declared
as its state: more specifically, here, the `hashmap` variable.

Inside this closure, we code how to react to the presence of an input batch or
a notification. In case of an input batch we get our chance to register to
the notification bus that we need to know the end of the epoch matching
the data coming in. We are calling notify_at repeatedly. It is a bit
offsetting, but the cost is amortized by the framework. It is preferable
to guessing whatever `to_stream()` does and it's what the timely guys
recommend anyway.

Moving on to the "payload" logic, we update the state hashmap. The
update_hashmap acts by inserting a new key or updating the value of an
existing key using the additioner we provide.

Second bit of logic is what to do when the notification occur: we will
send in the outgoing stream the sole count of entries in our map. This is where
we are doing half of the count. A strict implementation of reduceByKey would
shove the whole HashMap in the pipe, but we know better.

This is an instance of a Map being, again, the good guy.
Remember the pseudo-spark implementation above? It featured a
`reduceByKey(...).count()`. So the stream between these two logical operators
is a stream of disjoints HashMap. Count, on the other hand, could be
implemented as follow (again, pseudo-code).

```scala
    .map( h:HashMap => h.count() )
    .map( n => (0, n) )
    .reduceByKey((a,b) => a+b)
```

The first map calls .len() on the HashMap, the second make an absurd constant
key to bring all the partial counts to the same worker, which will sum to get
the results.

So what I have done here was to move "up" the first of the count map as close
as possible to the reduceByKey above, avoiding making a stream of HashMap and
having to explain that to rust.

Back to the code, everything that remains is to move everybody to a worker
(let's pick worker 0) and sum the numbers incoming.

```rust
let _: Stream<_, ()> = group.unary_notify(Exchange::new(|_| 0u64),
      "count",
      vec![],
      move |input, _, notif| {
          input.for_each(|time, data| {
              notif.notify_at(time);
              for x in data.drain(..) {
                  sum += x;
              }
          });
          notif.for_each(|_, _| {
              if index == 0 {
                  println!("result: {}", sum);
              }
          });
      });
});
```

Exchange is now a constant function, the rest is similar: the input chunks
contains numbers that are added to the worker-scoped `sum` variable. We
register the notification in the input handler, as we know spamming it is
tolerable. Once we get notified, we output the count on the console.

And that's it.

This executable is ready for the distributed mode, but the distribution
itself is not
covered: something or someone needs to copy the executable on each of the
cluster nodes, provide it the list of its peers and start it. 
If you want to run a second time you need
to go to each node and start the process again...
For the ec2 tests, I used ansible scripts.

## Recap

Of course, we are far from the elegant four lines of Spark, we had to do many
things by hand including translating the logical query to a distributable
executable plan. We produced a full screen with a good quantity of boilerplate,
but we have a standalone executable that will crunch the data with an
efficiency way higher than that of Spark or RedShift.

Building an equivalent of Spark or RedShift on top of timely is a daunting
task. But there are intermediary objectives that could make sense: helpers and
ready-to-use operators to make writing data processing task easier, daemons
that would migrate executables to all the nodes, start them and manage
the peer list for instance.

Not sure what's the next step is yet.

{% include {{site.baseurl}}/BigDataSeries.md %}
