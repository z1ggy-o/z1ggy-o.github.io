# Read as Needed: Building WiSER, a Flash-Optimized Search Engine


## Short Summary {#short-summary}

This paper proposed a NAND-flash SSD-friendly full text engine. This engine can
achieve better performance than existing engines with much less memory requestsed.

They reduce the unnecessary I/O (both the number of I/O and the volume). The
engine does not cache data into memory, instead, read data every time when query
arrive.

They also tried to increase the request size to exploit SSD internal parallelism.


## What Is the Problem {#what-is-the-problem}

Search engines pose great challenges to storage systems:

-   low latency
-   high data throughput
-   high scalability

The datasets become too large to fit into the RAM. Simply use RAM as a cache
cannot achieve the goal.

SSD and NVRAM can boost performance well. For example, flash-based SSDs provide
much higher throughput and lower latency compared to HDD.
However, since SSDs exhibit vastly different characteristic from HDDs, we need
to evolve the software on top of the storage stack to exploit the full potential
of SSDs.

In this paper, the authors rebuild a search engine to better utilize SSDs to
achieve the necessary performance goals with main memory that is significantly
smaller than the data set.


## Why the Problem Is Interesting {#why-the-problem-is-interesting}

There are already many studies that work on making SSD-friendly softwares
whithin storage stack. For example, RocksDB, Wisckey (KV-store); FlashGraph,
Mosaic (graphs processsing); and SFS, F2fS (file system).

However, there is no optimation for full-text search engines.
Also the challenge of making SSD-friendly search engine is different with other categories of
SSD-friendly softwares.


## The Idea {#the-idea}

The key idea is: **read as needed**.

The reason behind of the idea is SSD can provide millisecond-level read latency,
which is fast enough to avoid cache data into main memory.

There are three challenges:

1.  reduce read amplification
2.  hide I/O latency
3.  issue large requests to explit SSD performance

(This work is focus on the data structure, inverted index. This is a commonly
used data structed in information retrieve system. There are some brief
introduction about the inverted index in the paper, and I do not repeat the content here.)

| Category                  | Techniques                         |
|---------------------------|------------------------------------|
| Reduce read applification | - cross-stage data grouping        |
|                           | - two-way cost-aware bloom filters |
|                           | - trade disk space for I/O         |
| Hide I/O latency          | adaptive prefetching               |
| Issue large requests      | cross-stage data grouping          |


### Cross-stage data grouping {#cross-stage-data-grouping}

This technique is used to reduce read amplification and issue large requests.

WiSER puts data needed for different stages of a query into continuous and compact
blocks on the storage device, which increases block utilization when
transferring data for query.
Inverted index of WiSER places data of different stages in the order that it
will be accessed.

Previous search engines used to store data of different stages into different
range, which reduces the I/O size and increases the number of I/O requets.


### Two-way Cost-aware Filters {#two-way-cost-aware-filters}

When doing phrase queries we need to use the position informantion to check the
terms around a specific term to improve search precision.

The naive appraoch is to read the positions from all the terms in the phrase
then iterate the position list.
To reduce the unnecessary reading, WiSER emploies a bitmap-based bloom filter.
The reason to use bitmap is to reduce the size of bloom filter. There are many
empty entries in the filter array, use bitmap can avoid the waste.

_Cost-aware_ means comparing the size of position list with that of the bloom
filters. If the size of position list is smaller than that of bloom filters,
WiSER reads the position list directly.

Two-way filters shares the same idea. WiSER chooses to read the smaller bloom
filter to reduce the read amplification.


### Adaptive Prefetching {#adaptive-prefetching}

Prefetching is one of the commonly used technique to hide the I/O latency.
Eventhough, the read latency of SSD is small. Compare to DRAM, the read latency
of SSD still much larger.

Previous search engines (e.g. Elasticsearch) use fix-sized prefecthing (i.e.
Linux readahead) which increases the read amplification.
WiSER defines an area called _prefetch zone_. A prefetch zone is further divided
into prefetch segments to avoid accessing too much data at a time. It prefetches
when all prefetch zones involved in a quey are larger than a threshold.

To enable adaptive prefetch, WiSER hides the size of the prefetch zone in the
highest 16 bits of the offset in TermMap and calles `madvise()` with the
`MADV_SEQUENTIAL` hint to readahead in the prefetch zone.


### Trade Disk Space for I/O {#trade-disk-space-for-i-o}

Engines like Elasticsearch put documents into a buffer and compresses all data
in the buffer together.
WiSER, instead, compresses each document by themselves.

Compressing all data in the buffer together achieves better compression.
However, decompressing a document requires reading and decompressing all
documents compressed before the document, leading to more I/O and computation.
WiSER trades space for less I/O by using more space but reducing the I/O while
processing queries.

In the evaluation, WiSER uses 25% more storage than Elasticsearch. The authers
argue that the trade-off is acceptable in spite of the low RAM requirement of WiSER.

