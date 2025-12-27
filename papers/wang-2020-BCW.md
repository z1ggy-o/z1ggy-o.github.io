---
title: "FAST'20 - BCW: Buffer-Controlled Writes to HDDs for SSD-HDD Hybrid Storage Server"
date: 2023-09-17 12:51:01
permalink: /pages/237d08/
categories:
  - Notes
  - PaperNotes
tags:
author: [Me]
  # link: https://github.com/z1ggy-o
---

## Short Summary {#short-summary}

HDDs are under utilized in hybrid cloud storage systems which makes SSD to handle most of
the requests.
This shorts the life of SSDs and also wants the utilization of HDDs.

The authors of this paper find that the write requests can have $&mu;$s-level latency when
using HDD if the buffer in HDD is not full.
They leverage this finding to let HDD to handle write requests if the requests can fit into
the in disk buffer.

This strategy can reduce SSD pressure which prolongs SSD life and still provide relative good
performance.


## What is the problem {#what-is-the-problem}

In hybrid storage system (e.g. SSDs with HDDs), there is an unbalancing storage utilization problem.

More specifically, SSDs are used as the cache layer of the whole system, then data in SSDs is
moved to HDDs. The original idea is to leverage the high performance of SSD to serve consumer
requests first, so consumer requests can have shorter latency.

However, in a real system, SSDs handle most of the write requests and HDDs are idle in more
than 90% of the time. This is a waste of HDD and SSD because SSD has short life limitation.
Also, deep queue depth makes requests suffering long latency even when we using SSDs.


## Why the problem is interesting (important)? {#why-the-problem-is-interesting--important}

The authors find a latency pattern of requests on HDD, which is introduced by the using of the in disk buffer.
The request latency of HDD can be classified as three categories: _fast, middle_, and _slow_.
Write requests data is put to the buffer first, then to the disk. When the buffer is full,
HDD will block the coming requests until it flushes all the data in the buffer into disk.
When there are free space in the buffer, request latency is in fast or middle range, otherwise
in slow range.

The fast and middle latency is in $&mu; s$-level which similar with the performance of SSD.
If we can control the buffer in disk to handle requests which their size is in the buffer
size range, then we can get SSD-level performance when using HDD to handle small write
requests.


## The idea {#the-idea}

Dynamically dispatch write requests to SSD and HDD, which reduces SSD pressure and also
provides reasonable performance.

To achieve the goal, there are two key components in this paper:

-   Make sure requests to HDD are in the fast and middle latency range
-   Determining which write requests should be dispatch to HDD

To handle the first challenge, the authors provided a prediction model. The model itself
is simply comparing the current request size with pre-defined threshold.
We cannot know the write buffer size of HDD directly. However, we can get an approximate
value of the buffer size through profiling. The threshold are the cumulative amount of written data for the
fast/mid/slow stages.

Since we only want to use the fast and middle stages, we need to skip the slow stage.
There are two methods to do this. First, `sync` system call from host can enforce the
buffer flush; second, HDD controller will flush the buffer when the buffer is full.
`sync` is a expensive operation, so the authors choose to use _padding data_ to full fill
the buffer, which can let controller to flush the data in the buffer.

The second reason of why we need padding data is we want to make sure the prediction model
working well. That means the prediction model needs a sequential continuous write requests.
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
We do profiling to find the relation between SSD's queue depth and the request request latency.
In a certain queue depth, the request latency on SSD will be greater than the latency of HDDs
fast stage. We use the queue depth value as the threshold.
When queue depth exceeds the threshold, the system stores data to the HDD instead of SSD.


## Drawbacks and personal questions about the study {#drawbacks-and-personal-questions-about-the-study}

-   Only works for small size of write requests
-   The consistency is not guaranteed
-   The disk cannot be managed as RAID (can we?)
-   GC is still a problem
