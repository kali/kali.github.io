---
layout: post
title: Input formats
---

## I/O and CPU

We have seen in part #1 that my laptop is processing 30GB of deflated CSV in about
11 minutes. If we want to do better, the first step is find out what is our
bottleneck.

For years, we have worked under the assumption that IO where the limiting
factor in data processing. With SSD and PCIe disks, all this has changed.
Believe me, or re-run the bench and look at `top`, it's very obvious that we
are now CPU-bound.

Another simple way to convince us about this is to check out at what speed we
can read our 30GB data:

```
11:35 ~/dev/github/dazone% cat data/text-deflate/5nodes/uservisits/* | pv > /dev/null
  29GiB 0:00:46 [ 633MiB/s] [                                <=>         ]
```

If you did not know about `pv`, well, check it out. What we learn here is that
we manage very easily to read the files for our benchmark from disk in
46 seconds, at more than half a Gigabyte per second.

So we definitely have to look at what the CPUs are doing.

<img src="/assets/2016-02-01-Instruments.png" alt="profile, showing gz and csv" height="400px">

Without trying to decipher too much out of it, the four or five top lines show
a lot a time is spent in CSV and Zlib functions. More "decoding stuff" appears
just above the hilighted line (which is the actual final HashMap
construction and is below 3% of the computing resources consumed).

Not a huge surprise.

Compression makes a lot of sense when I/O bandwidth is scarce and
computing power is plentiful. But SSDs really altered the equilibrium
here (0.6 GB/s on a laptop...).

As for CSV, it is a human friendly format. It's relatively easy to parse by
eye (mostly due to the fact that each record is on its separate line).
But it is an absolute horror to parse for a computer. As a matter of fact,
most of the widely used data formats in data science are human readable,
and terrible for performance: CSV, Json, XML.

It is still frown upon to use a binary format, but we will probably have to get use
to it. Text formats are for Debug mode. HTTP is moving from a text format to
a Binary format. We have been doing HTTP in Debug mode for 20 years, this is
probably enough. Let's grow up, develop tools to pretty print these new
formats and protocols, and stop wasting so much computing power. Maybe we will
save battery time and polar bears.

Note that, as we try to find a better compromise for efficient data loading,
we do not necessarily want to change the way data is put in long-term cold
storage. It may be worth vaulting the data in highly compressed (bzip2) Json
or XML or whatever format will be felt resilient to technology evolution, and
use a separate, very loadable, probably binary, format to feed day-to-day
data analysis.

## Parser-friendly encoding

First, there is now a wide variety of binary Json-equivalent formats. They are
schema-free, support string, numbers, boolean, and some form of list and map.
BSON, MessagePack, bincode, CBOR happen to all have a ready-to-use rust
encoder and
decoder, so we can consider them as an alternative. It is relatively easy
to design and implement these kind of formats, so there are many competing
options today, and it's a bit hard to guess which formats will still be
around in a few years.

## ProtoBuf and Cap'n Proto

Then, we have schema-full formats. They require the developer to write a
format specification (think IDL) that the encoder and decoder will use
to read and write the data, freeing it of the redundancy that comes with
schema-free formats. There is a long and sometimes dark history here (yes,
CORBA, I'm looking at you), so some people will get uncomfortable at the
simple though of IDL description files. This is a bit of a shame.
In the past decade, new formats have emerged from big organizations (Facebook
and Google mostly) and were deemed relevant enough to get some traction
in open sources communities. They were originally mostly targeted at message
exchanging, but nothing prevents using them as storage formats. I have found
off-the-shelf Rust support for Protocol Buffers, Thrift, and
implemented a Protocol Buffers alternative in the bench.

ProtoBuf implemetation is a departure from the POD approach: the IDL
defines its own data class, where each field is actually a placeholder:
native scalar values are wrapped in `Option<>` and String are wrapped
in a proprietary wrapper. So switching to ProtoBuf implies a bit more work
than RustEncoding or Serde encoders.

[Cap'n Proto](https://capnproto.org/) is a relatively recent development,
"infinitely faster" than Protobuf. It does
not use POD or near-POD `struct`, but rather provides generated `Reader`
and `Builder` that wrap a buffer to provide accessor to the actual data.
The buffer encoding does not contain pointer, or architecture dependent
stuff, so it's ready to be send,read, written or shared. No encoding
or decoding when loading a record. This is particularly relevant in our
case because we only read one string from the record among seven.

Here again, we have to go through a proprietary interface to access the
data, so the switch is not completely trivial.

## Compression

Zlib-based formats are not the only option. Snappy and LZ4 may be relevant
alternatives. Not compressing the data at all may also be an option, actually.
I have not considered bzip2 as its computing requirement are even bigger than
Zlib's.

Plugging compression algorithms in and out of my code
has been relatively painless, except for a few snags that I think are
mostly due to the relative young age of the ecosystem: Rust having been
stable for less than one year, library implementors are still working
without a complete framework of good practise rules. Rust will get
there with time, experience and discussion.
