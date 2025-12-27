---
title: "VLDB'14 - Staring into the abyss: An evaluation of concurrency control with one thousand cores"
categories: 
  - paper notes
tags: 
  - DBMS
toc: false
isCJKLanguage: false
date: 2022-07-01 12:21:38
draft: false
permalink: /pages/cf5f1f/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

## Motivation

We are moving to the many-core architecture era, however, many design of database systems are still based on optimizing of single-threaded performance.
	
To understand how to design high performance DBMS for the future many-core architecture to achieve high scalability, addressing bottlenecks in the system is necessary.
	
This paper focus on concurrency control schemes.
	(spoiler:  the [following research]([[@An empirical evaluation of in-memory multi-version concurrency control]]) of [[Andrew Pavlo]] found out concurrency control schemes are not the most important component that affect the scalability of main memory DBMS on many-core environment )
	
## Contribution

- Evaluation of the scalability of seven commonly used concurrency control schemes (OLTP, in-memory DBMS)
- Evaluation is processed on a simulated machine with 1000 cores (actually is a cluster of 22 real machines)
	
## Solution

- This is evaluation paper, thus there is no solutions but evaluation analyses.
- All concurrency control schemes fail to scale to a large number of cores. The bottlenecks are:
	1. lock thrashing
	2. preemptive aborts
	3. deadlocks
	4. timestamp allocation
	5. memory-to-memory copying

- Lock thrashing happens in any waiting-based algorithm. Using the "non-waiting" can alleviate this problem, however, we will have more aborting.
	
- In high contention workloads, a non-waiting deadlock prevention scheme is better than deadlock detection. (Restart is fast in main memory DBMS)
- Memory allocation (includes timestamp allocation) is usually managed by a centric data structure, which becomes the bottleneck. Avoiding shared, centric data structure is important to achieve higher scalability.
- Add new hardware to offload some tasks (e.g., memory copying) from CPU is a feasible way to improve the scalability
	
## Evaluation
- The evaluation platform is based on a CPU simulator, Graphite (from MIT).
- The authors implemented a lightweight main memory DBMS only for this testing.
- Workloads are YCSB and TPC-C
	
## Main finding of the paper
- Like other systems, to improve the scalability, how to avoid shared data is the key.
- "Measure, Then Build", and a thorough measurement is always needed. This paper is good enough on analyzing concurrency control schemes, however, the following research revels that this is not the most important part...
