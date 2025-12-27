---
title: "OSDI'04 - MapReduce: simplified data processing on large clusters"
categories: 
  - paper notes
tags: 
  - distributed-systems
toc: false
isCJKLanguage: false
date: 2022-05-16 09:44:18
draft: false
permalink: /pages/ecb45c/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

## MOTIVATION

Google does a lot of straightforward computations in their workload. However, since the input data is large, they need to distribute the computations in order to finish the job in a reasonable amount of time.

To parallelize the computation, to distribute the data, and to handle failures make the original simple computation code complex. Thus, we need a better way to handle these issues, so the programmer only needs to focus on the computation task itself and does not need to be a distributed systems expert.

## CONTRIBUTION

They implement a library that can provide a simple interface that enables automatic parallelization and distribution of large-scale computations. In addition, the library handles machine failures without interaction with programmers.

## SOLUTION

- Inspired by the `map` and `reduce` primitives present in functional languages, they provide a restricted programming model that parallelizes the computation automatically (the computation must be deterministic).
- Both Map and Reduce functions are written by the user. The input data will be partitioned into M splits, and each Map task handles one of them. The intermediate output of the Map function is divided into R pieces. The Reduce task will read the intermediate output of the Map function and generate the final output files.
- For a cluster of machines, we have only one master machine, and the others are worker machines. The master machine assigns tasks (Map or Reduce) to workers and tracks the state of each task. The worker machines communicate with the master when work is finished or communicate with other workers to read data to process.
- Fault tolerance is handled by periodical communication and re-computation when a machine fails. Because the computation is deterministic, recomputing the same task is okay.
- Many optimizations are applied in their implementation. Check the paper for details.

## EVALUATION

They used a cluster that consisted of around 1800 machines. Each machine has only 4GB of memory, and two 160GB IDE disks with a gigabit Ethernet link.

They tasted grep and sort workloads with 10^{10} 100-byte records (around 1TB). The results are shown in the terms of data throughput with the timeline.

## MAIN FINDING OF THE PAPER

It is useful to abstract a common pattern of certain computing tasks, and create an infrastructure to handle the common issues.
