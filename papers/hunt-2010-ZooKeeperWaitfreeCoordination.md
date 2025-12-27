---
title: "ATC'10 - ZooKeeper: Wait-free Coordination for Internet-scale Systems"
categories: 
  - paper notes
tags: 
  - distributed-systems
toc: false
isCJKLanguage: false
date: 2022-05-23 09:00:39
draft: false
permalink: /pages/c0be75/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

## Motivation

Large-scale distributed applications require different forms of coordination.

Usually, people develop services for each of the different coordination needs. As a result, developers are constrained to a fixed set of primitives.

## Contribution

- Exposes APIs that enables application developers to implement their own primitives, without changes to the service core.
- Achieve high performance by relaxing consistency guarantees

## Solution

ZooKeeper provides to its clients the abstraction of a set of data nodes (znodes). These data nodes are organized like a traditional file system.

ZooKeeper provides a set of simplified APIs to access these data nodes. Application develops can use these APIs to implement different forms of coordination that they want.

ZooKeeper servers use an atomic broadcast protocol, Zab (similar to Raft) to sync state between each other.

It is vague about how to implement ZooKeeper. However, since it is an open source project, the source code is always available.

In short, ZooKeeper guarantees correct coordination with high performance through:
- Using wait-free data objects instead of blocking.
- FIFO client ordering of all operations.
- Linearizing all writes to the leader, then propagate to other servers (they call this A-linearizability).
- Read locally. Can have stale data, however, much faster.
- Asynchronous enables batching to reduce networking and storage I/O overhead.

## Main finding of the paper

- Abstraction of basic building components can help create more general systems.
- ZooKeeper is one of the best examples that shows us how to weak consistency in favor of higher performance.
