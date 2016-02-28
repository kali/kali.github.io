---
layout: post
title: One look at query3
---

```sql
SELECT sourceIP, totalRevenue, avgPageRank
FROM ( SELECT sourceIP,
              AVG(pageRank) as avgPageRank,
              SUM(adRevenue) as totalRevenue
         FROM Rankings AS R, UserVisits AS UV
        WHERE R.pageURL = UV.destURL
          AND UV.visitDate BETWEEN Date('1980-01-01') AND Date('X')
     GROUP BY UV.sourceIP )
ORDER BY totalRevenue DESC LIMIT 1
```

I dare you to look at it for less than thirty seconds, close your eyes
and remember what it does. I hate SQL syntax. Let's try to get this in
some operational order:

* discard UserVisits not in the right date interval
* join it with rankings (by the url)
* group that by sourceIP, summing adRevenue and averaging pageRank
* sort it totalRevenue
* return the first triplet

Note there is huge waste in the last two steps: what we return is
maximum but we order the whole thing to get it. This is absurd, but
there is no elegant way in standard SQL to get the full first row of a
relation using one column as a sort key. So you have to write this, and hope
the optimizer is smarter than the language.

Of course sorting to implement a
[selection](https://en.wikipedia.org/wiki/Selection_algorithm) is bad if the
data fits in memory, but it gets even worse when it does not.

## Query 3 on the laptop

Query2 was relatively easy: we scanned one input table
(UserVisits) while building a result in a big HashMap that could, with some 
care, fit in the memory of a (big) laptop. Query 3 is another kind of beast.

* UserVisits
    * 2037 chunks
    * about 800,000 records per chunk
    * for each chunk source IP weight 8MB, destURL 25MB
* Rankings
    * 100 chunks
    * 900,000 records per chunk
    * urls: 56MB per chunks
* Results (according to the original benchmark)
    * 485,312 to 533,287,121 rows depending on X
    * each result is a source IP, plus two single precision numbers

So, Rankings is big, but *may* fit in the laptop memory:
56MB (urls) + 4MB (900k*4, the pageRank) * 100 ~ 6GB. If we make sure sure the
HashMap behaves, we should be fine.

It's interesting,
because it helps implementing the join operation if one of the relations can
be put in a hash in memory:

* load the small one in a hashmap keyed by the join field
* scan the big relation
    * for each record lookup the hashmap
    * if there is a match output the composite record

Now, there is the issue of the sort. Contrary to what happened in Query2, we do
not aggregate by a IP prefix, but by the full field. This explains why this
query yields so much more lines than Query2, even if we have the filtering by
date. On the laptop, there is no way we can build this in RAM.
