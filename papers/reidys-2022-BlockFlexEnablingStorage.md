---
title: "OSDI'22 - BlockFlex: Enabling Storage Harvesting with Software-Defined Flash in Modern Cloud Platforms"
categories: 
  - paper notes
tags: 
  - storage-systems
  - cloud-computing
toc: false
isCJKLanguage: false
date: 2022-08-07 11:39:27
draft: false
permalink: /pages/a95802/
author: Gy
---

## Motivation

- Cloud computing system is high virtualised for higher resource utilization. 
  Recently, cloud providers have developed harvesting techniques to allow evictable virtual machines to use unallocated resources.  

- People mostly focus on how to use unallocated computing resources and memory resources, however, there is no harvesting techniques that built for storage resources.

- This paper proposes a harvesting framework for storage resources that provides isolation of space, bandwidth, and data security.


## Contribution

- They conduct a characterization study of the storage efficiency in different cloud platforms through trace logs provided by these platforms.
    
- They come up a new abstraction of storage virtualization for software-defined SSDs. And base on that, they build a learning-based storage harvesting framework, BlockFlex.
    

## Solution

- First, they analyze the tracks provided by Alibaba and Google and find out that there are a lot of unused storage resources (both in capacity and bandwidth) in allocated and unallocated VMs. That’s the resource that we can use for evictable VMs temporarily.
    
- A SSD consists of multiple channel internally. Each channel has its own storage chips and can operate independently. In the context of SSD virtualization, software-defined SSD (SDF) allows the upper-level VM to map its virtual SSD (vSSD) to a set of flash channels. That’s how we control capacity and bandwidth allocation.
    
- BlockFlex proposes a new abstraction, ghost vSSD (gSSD), on top of SDF, attached to existed vSSD. A gSSD is represented by a data structure that records its bandwidth, capacity, expire time etc. gSSDs are organized by a set of linked list, each linked list contains gSSDs with bandwidth and capacity in a specific range. Because the allocation/deallocation does not happen frequently, this organization is enough for finding appropriate gSSDs efficiently.
    
- Create gSSDs from unsold (unallocated) VMs
    
    - Because the storage usage of unsold VMs is quit stable. BlockFlex tags each unsold VM with a heuristic-based predicated duration time using the histogram of previous available times for the unsold VM with the same storage capacity.
        
- Create gSSDs from sold VMs
    
    - Since sold VMs are currently using by clients, the heuristic-based prediction does not work well.
        
    - BlockFlex allocates read, write, and erase operations at the block layer of each vSSD for online predictions. They choose LSTM as the models because they can do time-series predictions. The operation tracks are used to infer the bandwidth, throughput (IOPS), and current storage utilization, then these info is used as input for LSTMs.
        
    - There are bandwidth predictor, capacity predictor, and duration predictors. Both bandwidth predictor and capacity predictor use the same LSTM model, but with different learning rate and hidden layer size. The predicted bandwidth and capacity are then used as the input for the duration predictors.
        
- Reclaim gSSDs
    
    - Upon the gSSD reclamation, the corresponding entries in the address mapping table of the vSSD will be removed.
        
    - If a gSSD will expire soon, BlockFlex will erase the flash blocks for data safety, and remove the gSSD instance.
        

## Evaluation

- The workloads they used are TeraSort, ML Prep, PageRank, and YCSB. The capacity is ranged from around 60GB to 220GB (not much)
    

## The main takeaway

- It seems people trying to give more control of storage devices to the host side instead of using them as a black box.
