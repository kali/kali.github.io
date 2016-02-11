---
layout: post
title: Embrace the glow cloud
---

This is part #5 of a series about a BigData in Rust experiment.

We are working on a simple query nicknamed Query2 and comparing
our results to the
[BigData benchmark](https://amplab.cs.berkeley.edu/benchmark/).

```SQL
  SELECT SUBSTR(sourceIP, 1, X), SUM(adRevenue)
    FROM uservisits
GROUP BY SUBSTR(sourceIP, 1, X)
```

| Query   |  X | group count | Hive | Shark | Redshift | My laptop |
|:-------:|---:|------------:|-----:|------:|---------:|----------:|
|   2A    |   8|   2,067,313 | 730s |   83s |      25s |   **50s** |
|   2B    |  10|  31,348,913 | 764s |  100s |      56s |   **64s** |
|   2C    |  12| 253,890,330 | 730s |  132s |      79s |   **91s** |

The last column is what we manage to get on my laptop using Rust, with
quite a lot of work and
trying to stay in the realm of "fair" optimisations that an automatic query
planner could find.

The other column are partial results from the BigData benchmark, running on
a 8$/hr on Amazon EC2.

So let's do the same. Let's go to EC2 and buy us a 8$/hr cluster.

# Timely dataflow

My implementation of Query2 is not distributable. It is multithreaded, but
can not span process or hosts. Doing this by hand would be a lot of work, but
the Rust ecosystem provides a little gem called Timely dataflow.

Timely dataflow is designed as a low-level toolbox for distributing work. To
quote the README, it is a "low-latency cyclic dataflow computational model".
Frank McSherry experimented with the concept of timely dataflow while working
on Naiad at Microsoft Research, and has since ported it from C# to Rust.

Integrating Query2 in timely has not been very difficult — even if I had to 
get some help from higher deities — but induced a change in the
general algorithm which I have described in the previous post.

It turns out the way timely lead me to write the aggregation is about 10% more
efficient than my ad-hoc implementation, so I won't complain. I have not
completely explored the reason that make is so. Re-organising the ad-hoc
implementation so that the data flows in a way similar to the way it flows
in the timely program may help.

Anyway. Let's go to the ec2 pricing page. The sky is the limit, up to 8$/hr.

## Results on EC2 cluster

EBS disks tends to make things less predictable, so we will start with
instances having enough SSD as local storage. We want to try "work-horse"
instances (m series) but as we have established we need to care about CPU,
we also want to try instances from the C series.

So eligible are m3 and c3. m3.medium and m3.large have not enough disk space,
so the cheapest possible M choice is m3.xlarge. This instance is $0.266 per
Hour, so we would need 30 of them to get to get to the 8$/hr mark.

I setup 20 instances with stripped RAID on the two SSD disks to handle the
data. A hdparm check shows a bandwidth similar to the laptop, a bit higher,
between 750MB/s to 1000MB/s, depending on the actual instance.

Each host run Query2 in single host timely mode first (at the 0 node mark),
then we run the process in clustering mode, including more and more hosts, up
to 20.

m3.xlarge   4   13  15  2 x 40 SSD  $0.266 per Hour

<img    alt="Performance of timely running Query2 on 1 to 20 m3.xl instances"
        src="{{site.baseurl}}/assets/2016-02-11-m3xl.png"/>

## Other choices of instances

m3.2xlarge  8   26  30  2 x 80 SSD  $0.532 per Hour


<img    alt="Performance of timely running Query2 on 1 to 20 m3.2xl instances"
        src="{{site.baseurl}}/assets/2016-02-11-m32xl.png"/>

c3.8xlarge  32  108 60  2 x 320 SSD $1.68 per Hour

placed in the same placement group

<img    alt="Performance of timely running Query2 on 1 to 20 c3.8xl instances"
        src="{{site.baseurl}}/assets/2016-02-11-c38xl.png"/>

