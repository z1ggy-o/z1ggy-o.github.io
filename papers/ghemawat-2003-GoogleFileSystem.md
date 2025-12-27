---
title: "SOSP'03 - The Google file system"
categories: 
  - paper notes
tags: 
  - distributed-systems
toc: false
isCJKLanguage: false
date: 2022-05-22 11:11:56
draft: false
permalink: /pages/a16968/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

## MOTIVATION

Google needs a new distributed file system to meet the rapidly growing demands of Google’s data processing needs (e.g., the MapReduce).

Because Google is facing some technical challenges different with general use cases, they think it is better to develop a system that fits them well instead of following the traditional choices. For example, they chosen to trade off consistency for better performance.

## CONTRIBUTION

They designed and implemented a distributed file system, GFS. This system can leverage clusters consisted with large number of the machines. The design puts a lot of efforts on fault tolerance and availability because they think component failures are the norm rather than the exception.
	
This system is developed only for Google’s own programs. As a result, GFS does not provide POSIX APIs. Programs are designed and implemented based on GFS, which simplifies the design of GFS.

## SOLUTION
    
- GFS uses a single master multiple chunkservers architecture. The master maintains all file system metadata and the chunkservers handle the file data. The master periodically communicates with each chunkserver in HeartBeat messages to give it instructions and collect its state.
- Considering the characteristics of the workloads in Google (append only, sequential read), they decide to divide files into fixed-size chunks (64MB). Each chunk has a globally unique chunk handle assigned by the master, that’s how we can find a chunk of a specific file.
- The client gives file name and in file offset to the master. Then, the master will send back the corresponding chunk handle and the chunkservers that have that chunk. After that, clients will communicate with chunkservers directly. This approach avoids the single master becoming the bottleneck.
- To ensure high availability and also improve parallelism, each file chunk is replicated on multiple chunservers on different racks. The metadata in master is protected by the operation log, also this log is replicated on multiple machines.
- When write happens, the data mutation propagates along the chunkservers incrementally. As a result, the write becomes faster, but clients can read stale data.

## EVALUATION

- Micro-benchmarks on a small cluster with 16 chunkserver. Tested the read, write, and record append performance.
- Two real world clusters in Google. One for production, another one for research and development.

## LIMITATION

With the increasing volume of data, the single master design can no longer cope with the demands.

## MAIN TAKEAWAY

There are times when we can discard generalization and design a dedicated system for a specific scenario, which leads to a more simply design.
