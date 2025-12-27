---
title: "VLDB'17 - An empirical evaluation of in-memory multi-version concurrency control"
categories: 
  - paper notes
tags: 
  - DBMS
toc: false
isCJKLanguage: false
date: 2022-07-01 12:36:05
draft: false
permalink: /pages/240e04/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

## Motivation  
- MVCC is the most popular scheme used in DBMSs developed in the last decade. However, there is no standards about how to implement MVCC.  
- Many DBMSs tell people that they use MVCC and how they implement it. There are several design choices people need to make, however, no one tells why they choose such way to implement their MVCC.  
- This paper gives a comprehensive evaluation about MVCC in main memory DBMSs and shows the trade-offs of different design choices.  

## Contribution  
- A good introduction about MVCC.  
- Evaluation about how concurrency control protocol, version storage, garbage collection, and index management affect performance on a real in-memory DBMS.  
- Evaluation of different configurations that used by main stream DBMSs.  
- Advisings about how to achieve higher scalability.  

## Solution  
- This is an evaluation paper. There is no "solution" but evaluation analyses.  
- People mainly focus on concurrency control protocols when they talk about scalability. However, the evaluation results show that the version storage scheme is also one of the most important components to scaling an in-memory MVCC DBMS in a multi-core environment.  
- Delta storage scheme is good for write-intensive workloads especially only a subset of the attributes is modified. However, delta storage can have slow performance on read-heavy analytical workloads because it spends more time on traversing version chains.  
- Most DBMSs choose to use tuple-level GC. However, the evaluation result shows that transaction-level can provide higher performance (at least in main memory DBMS) because it has smaller memory footprint.  
- In terms of index management, logical pointer is always a better choice than physical pointers.  
- The design choices that Oracle/MySQL and NuoDB made seems have the best performance.  

## Evaluation  
- They uses a DBMS implemented in CMU, called Peloton.  
- The evaluation platform has 40 cores with 128 GB memory.  
- Workloads are YCSB and TPC-C (both OLTP)  

## The main finding of this paper  
- If you want to learn MVCC, read this one.  
- Still, "Measure, Then build".  
