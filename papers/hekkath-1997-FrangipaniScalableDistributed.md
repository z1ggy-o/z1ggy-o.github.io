---
title: "SOSP'97 - Frangipani: a scalable distributed file system"
categories: 
  - paper notes
tags: 
  - distributed-systems
toc: false
isCJKLanguage: false
date: 2022-07-01 13:30:21
draft: false
permalink: /pages/cd2688/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---
 
## Motivation  
- File system administration for a large, growing computer installation is a laborious task.  
- A scalable distributed file system that can handle system recovery, reconfiguration, and load balancing with little human involvement is what we want.

## Contribution  
- A real system that shows us how to do cache coherence and distributed transactions.  


## Solution  

- The file system part is implemented as a kernel module, which enables the file system to take advantage of all the existing OS functionalities, e.g., page cache.  
- Data is stored into a virtual disks, which is a distributed storage pool that shared by all file system servers.  
- All file system operations are protected by a lock server. Client will invalid the cache when it release the lock. As a result, client can only see clean data and the cache coherence is ensured.  
- Because the client decides when to give the lock back, Frangipani can provide transactional file-system operations. (take locks on all data object we need first, only release these locks when all operations are finished.)  
- Write-ahead logging is used for crash recovery. Because all the data are stored in the shared distributed storage, the log can be read by other servers and recover easily.  
- The underlying distributed storage and the lock server are both using Paxos to ensure availability.  


## Evaluation  

- They evaluated the system with some basic file system operations, such as create directories, copy files, and scan files e.t.c.  
- They tested both the local performance and the scalability of the proposed system.  


## The Main Finding of the Paper  
- Complex clients sharing simple storage can have better scalability.  
- Distributed lock can be used to achieve cache coherence and distributed transaction.  
