# 


## Paper notes {#paper-notes}


### <span class="org-todo done DONE">DONE</span> Read as Needed: Building WiSER, a Flash-Optimized Search Engine {#he-2020-ReadNeededBuilding}


#### Short Summary {#short-summary}

This paper proposed a NAND-flash SSD-friendly full text engine. This engine can
achieve better performance than existing engines with much less memory requestsed.

They reduce the unnecessary I/O (both the number of I/O and the volume). The
engine does not cache data into memory, instead, read data every time when query
arrive.

They also tried to increase the request size to exploit SSD internal parallelism.


#### What Is the Problem {#what-is-the-problem}

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


#### Why the Problem Is Interesting {#why-the-problem-is-interesting}

There are already many studies that work on making SSD-friendly softwares
whithin storage stack. For example, RocksDB, Wisckey (KV-store); FlashGraph,
Mosaic (graphs processsing); and SFS, F2fS (file system).

However, there is no optimation for full-text search engines.
Also the challenge of making SSD-friendly search engine is different with other categories of
SSD-friendly softwares.


#### The Idea {#the-idea}

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

<!--list-separator-->

-  Cross-stage data grouping

    This technique is used to reduce read amplification and issue large requests.

    WiSER puts data needed for different stages of a query into continuous and compact
    blocks on the storage device, which increases block utilization when
    transferring data for query.
    Inverted index of WiSER places data of different stages in the order that it
    will be accessed.

    Previous search engines used to store data of different stages into different
    range, which reduces the I/O size and increases the number of I/O requets.

<!--list-separator-->

-  Two-way Cost-aware Filters

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

<!--list-separator-->

-  Adaptive Prefetching

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

<!--list-separator-->

-  Trade Disk Space for I/O

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


### <span class="org-todo done DONE">DONE</span> BCW: Buffer-Controlled Writes to HDDs for SSD-HDD Hybrid Storage Server {#bcw-buffer-controlled-writes-to-hdds-for-ssd-hdd-hybrid-storage-server}


#### Short Summary {#short-summary}

HDDs are under utilized in hybrid cloud storage systems which makes SSD to handle most of
the requests.
This shorts the life of SSDs and also wasts the utilization of HDDs.

The authors of this paper find that the write requets can have $&mu;$s-level latency when
using HDD if the buffer in HDD is not full.
They leverage this finding to let HDD to handle write requests if the requests can fit into
the in disk buffer.

This strategy can reduce SSD pressure which prolongs SSD life and still provide relative good
performance.


#### What is the problem {#what-is-the-problem}

In hybrid storage system (e.g. SSDs with HDDs), there is a unbalancing storage utilization problem.

More specifically, SSDs are used as the cache layer of the whole system, then data in SSDs is
moved to HDDs. The original idea is to leverage the high performance of SSD to serve consumer
requests first, so consumer requests can have shorter lantecy.

However, in a real system, SSDs handle most of the write requests and HDDs are idle in more
than 90% of the time. This is a waste of HDD and SSD because SSD has short life limitation.
Also, deep queue depth makes requests sufferring long latency even when we using SSDs.


#### Why the problem is interesting (important)? {#why-the-problem-is-interesting--important}

The authors find a latency pattern of requests on HDD, which is introduced by the using of in disk buffer.
The request latency of HDD can be classified as three catagories: _fast, middle_, and _slow_.
Write requets data is put to the buffer first, then to the disk. When the buffer is full,
HDD will block the coming requests until it flushes all the data in the buffer into disk.
When there are free space in the buffer, request latency is in fast or middle range, otherwise
in slow range.

The fast and middle latency is in $&mu; s$-level which similar with the performance of SSD.
If we can control the buffer in disk to handle requests which their size is in the buffer
size range, then we can get SSD-level performance when using HDD to handle small write
requests.


#### The idea {#the-idea}

Dynamically dispatch write requests to SSD and HDD, which reduces SSD pressure and also
provides resonable performance.

To achieve the goal, there are two key components in this paper:

-   Make sure requests to HDD are in the fast and middle lantency range
-   Determing which wirte requests should be dispatch to HDD

To handle the first challenge, the authors provided a prediction model. The model itself
is simply comparing the current request size with pre-defined threshold.
We cannot know the write buffer size of HDD directly. However, we can get an approximate
value of the buffer size through profiling. The threshold are the cumulative amount of written data for the
fast/mid/slow stages.

Since we only want to use the fast and middle stages, we need to skip the slow stage.
There are two methods to do this. First, `sync` system call from host can enforce the
buffer flush; second, HDD controler will flush the buffer when the buffer is full.
`sync` is a expensive operation, so the authors choose to use _padding data_ to full fill
the buffer, which can let controller to flush the data in the buffer.

The second reason of why we need padding data is we want to make sure the prediction model
working well. That means the prediction model needs a sequencial continuous write requests.
When HDD is idle, the controller will empty the buffer even when the buffer is not full,
which break the prediction. Read requests also break the prediction.
Using padding data can help the system to maintain and adjust the prediction.
More specifically, when HDD is idle, the system use small size padding data to avoid disk
control flush the buffer; when read requests finished, since we cannot know if the disk
controller flushes the buffer, the system use large size padding data to quickly full fill
the buffer, which can help recorrect the prediction model.
These padding data will be remove during the GC procedure.

Steering requests to HDDs is much easier to understand. The latency of request is related
to the I/O queue depth.
We do porofiling to find the relation between SSD's queue depth and the request request latency.
In a certain queue depth, the request latency on SSD will be greater than the latency of HDDs
fast stage. We use the queue depth value as the threshold.
When queue depth exceeds the threshold, the system stores data to the HDD instead of SSD.


#### Drawbacks and personal questions about the study {#drawbacks-and-personal-questions-about-the-study}

-   Only works for small size of write requests
-   The consistency is not guaranteed
-   The disk cannot be managed as RAID (can we?)
-   GC is still a problem


### <span class="org-todo todo TODO">TODO</span> aghayev-2019-FileSystemUnfit {#aghayev-2019-filesystemunfit}


#### Short Summary {#short-summary}

This paper mostly consists of two parts. The first part tells us why the `FileStore` has performance issues.
And the second part tells us how Ceph team build `BlueStore` based on the
lessons that they learnt from `FileStore`.

The main ideas of `BlueStore` are:

1.  Avoid using local file system to store and represent Ceph objects
2.  Use KV-store to provide transaction mechanism instead of build it by ourself


#### What's the problem {#what-s-the-problem}

There is a software called `storage backend` in Ceph. The `storage backend` is
responsible to accept requests from upper layer of Ceph and do the real I/O on
storage devices.

Ceph used to use commonly used local file systems based storage backend (i.e.,
FileStore). FileStore works, but not that well. There are mainly three
drawbacks in FileStore:

1.  Hard to implement efficient transactions on top of existing file systems
2.  The local file system' metadata performance is not great (e.g., enumerating
    directories, ordering in the return result)
3.  Hard to adopt emerging storage hardware that abandon the venrable block
    interface (e.g., Zone divecies)


#### Why the problem is interesting {#why-the-problem-is-interesting}

1.  Because storage backends do the real I/O job, the performance storage
    backends domains the performance of the whole Ceph system.
2.  For years, the developers of Ceph have had a lot of troubles when using local
    file systems to build storage backends


#### The Core Ideas {#the-core-ideas}

<!--list-separator-->

-  The problems of FileStore

    -   **Transaction**: in Ceph, all writes are transactions. There are three options for

    providing transaction on top of local file systems: leveraging file system
    internal transaction; implementing the write-ahead logging (WAL) in user space;
    and using a key-value store as the WAL

    -   leveraging file system's transaction is not a good idea because they usually
        lack of some functionalities. And also, file system is in the kernel side, it
        is hard to expose the interfaces.
    -   User space WAL: has consistency problem (not atom operations, because it is
        a logical WAL, it use read/write system calls to write/read data to/from
        logging part). The cost for handle the problem is expensive. Also slow
        read-modify-write and double-write (logging all data) problems.
    -   Using KV-store: This is the cure. However, there is still some unnecessary
        file system overhead like _journaling of journal_ problem

    <!--listend-->

    -   **Slow metadata operations**: enumeration is slow. The read result from a object
        sets should in order, which file systems do not do. We need to do sorting
        after read. To reduce the sorting overhead, Ceph limits the number of files in
        a directory, which introduces _directory splition_. The dir splition has
        overhead.

    -   **Does not support new storage hardware**: new storage devices may need
        modifications in the existing file systems. If Ceph uses local file systems,
        the Ceph team can only wait for the developers of the file systems to adopt
        the new storage.

    In paper, there are more details. In summary, the reasons of above problems are:

    1.  File system overhead
    2.  Ceph uses file system metadata to represent Ceph object metadata. (i.e.,
        object to file, object group to diectory) The file system metadata operations
        are not fast and also may have some consistent issues.

<!--list-separator-->

-  BlueStore

    1.  Does not use local file system anymore. Instead, store Ceph objects into raw
        storage directly. This method avoids the overhead of file systems.
    2.  Use KV-store (i.e. RocksDB) to provide transaction and manage Ceph metadata.
        This method provides much faster metadata operations and also avoid building
        transtion mechanism by Ceph developer.
    3.  Because RocksDB runs on top of file systems. BlueStore has a very simply file
        system that only works for RocksDB called BlueFS. The BlueFS stores all the
        contents in logging space and cleans invalid data periodly.

    If you understand the reason of why FileStore performs not well, you can simply
    understand the choices they did when build BlueStore.

    BlueStore still has some issues. For example, because BlueStore do not use file
    systems, it cannot leverage the OS page cache and need to build the cache by
    itself. However, build a effective cache is hard.

