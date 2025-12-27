---
title: LevelDB 源码分析笔记
categories: 
  - articles
tags: 
  - DBMS
  - LSM-Tree
ShowToc: true
TocOpen: false
isCJKLanguage: true
date: 2022-08-22 13:30:19
draft: false
permalink: /pages/2f7119/
sidebar: auto
author: Gy
---

最近一周花了些时间阅读了 LevelDB 的代码，了解一下 LSM-Tree 的具体实现。在阅读的过程中简略的记录了一些笔记，放在了我的 [Github repo](https://github.com/z1ggy-o/LevelDB-Notes/tree/main/Notes) 里。笔记是用蹩脚英语写的，也没有做任何的插图，建议配合[此分析文档](https://leveldb-handbook.readthedocs.io/zh/latest/basic.html)一起食用。

LSM-Tree 虽然很早被提出，但是在 LevelDB 这个开源实现被公开之后才引发了比较广泛的研究。毕竟，~~拿来的是最舒服的~~，虽然概念简单，但是细节很多。有些时候代码比语言更清晰，所以，我的笔记里主要还是将概念和具体的代码做了关联，细节还是要到代码里去扣。

在这里，主要想要用简单的语言介绍一下 LevelDB 的主要框架。方便想要花五分钟大致了解一下 LevelDB 的朋友。

## LevelDB，一个 LSM-Tree 的开源实现

[LSM-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) 的特点在于其对存储内容的排序和 append-only 的写入模式。此工作模式主要有两个优点：
- 极佳的写入性能。不需要做 read-modify-write。
- 便利的并行控制。写入的内容不会再被修改，可以放心的做并行读取。

但是物理世界里，一切都是有利弊的。LSM-Tree 的读取性能并不优秀。因为有过期数据的存在，读取过程中可能会需要读取多个文件，会有 read-amplification 的问题。过期数据也会浪费存储空间。

插一句，对 write-amplification (WA) 问题，LSM-Tree 和 B-Tree 两者都有，但是具体的情景是不一样的。LSM-Tree 的 WA 来源于 compaction 过程，需要将已有的数据重新写入。而 B-Tree 的 WA 来源于其固定 page 大小的写入单位。就算一个 page 内只有一个 byte 被修改了，整个 page 也需要写回。

接下来主要针对 LevelDB 的写入和读取两个部分的设计部件作一下简介。

## 写入：Memtable 和 SSTable

LevelDB 只允许写入请求（即插入，更新，删除）发生在内存内的 memtable 中。Memtable 是实现于 skiplist 数据结构，在提供良好插入性能的情况下也保持 item 有序。

当一个 memtable 达到预定的大小之后，memtable 会被写入到磁盘中，成为一个 SSTable 文件。因为在 memtable 写出的时候就不能再接收写请求了，LevelDB 使用了 double-buffer 的策略，另起一个 memtable 接收写请求。

为了避免故障带来的数据损失，LevelDB 也使用了 WAL 的方式来记录请求内容。不过因为 LevelDB 只是一个简单的 KV 引擎，log 几乎就是请求内容的复制记录。而且，在 memtable 被持久化之后，对应的 log 文件就可以抛弃了。

总的来说，log-structuring 本身已经为写入带来了极大的便利，所以这个部分的特别设计并不多。

## 读取：SSTable 结构和 MVCC

因为 LSM-Tree 的读取性能并不好，LevelDB 为了提升读取性能还是花了一番力气的。

### SSTable 结构

Memtable 只是一个 skiplist，其中一个 node 代表一个写入请求指令的内容。如何将这些内容进行安排，决定了读取过程的效率。LevelDB 主要做了一下几个设计：
- SSTable 文件按新旧序号排列。更新的文件中自然存放着更新的数据。
- SSTable 文件的底部设计了一个 footer 部分，可以帮助快速识别目标 key 所在的位置，减少读取量。
- SSTable 内部嵌入 bloom filter，进一步减少不必要的读取。

更加具体的内容请参照笔记和源码。

### MVCC

LevelDB 是支持针对数据进行 snapshot 读取的。它的实现方式倒也简单直接。在 DB 的每一条记录中，都包含一个单调递增的 sequence number。当要进行 snapshot 读取的时候，LevelDB 会告诉读请求当时的最新 sequence number。如此一来通过比较，就可以避免读到更新的数值了。这个设计的好处在于，读不会阻塞写。

LevelDB 抽象了一个 `Version` 对象，记录了一个版本的文件。每当有 sstable 生成或删除的时候，就形成一个新的版本。如果有请求在使用某个版本的文件，那么即使这个文件已经过期，也会等到所有的使用都完毕之后才会被清理。

### Cache

除此之外，LevelDB 也实现了一个比较标准的 LRUCache。该 cache 主要缓存打开的文件句柄，以及读取的数据块。

用户也可以选择不使用这个 cache。比如，如果做大范围的 range read 的话，用 cache 的意义不大。

## 清理，Compaction

因为 append-only 的写入方式会让 DB 中存有过期的数据，我们需要定期的对数据进行清理，释放不需要的存储空间。

LevelDB 使用 leveling compaction 的方式来做这个清理。具体来说，sstable 文件被按层级安排在了一起，低层的数据存放量少，数据更新；高层的数据存放量更大，但是数据旧。

compaction 是在两个相邻层级中的文件中进行的。先是在上层中找到一个对象文件，然后根据其 key 的范围，在下一层中找到有 key 重叠的文件。最后，因为所有的记录都是排好序的，利用 k-way merge 的方式就可以将过期的数据筛除。有效的数据被写入到新的 sstable 文件中，放入到下层里。这个过程可能在各个层级递归进行。

寻找 compaction 对象文件的过程略微复杂，有兴趣可以参照笔记和源码学习。

因为 compaction 是一个比较昂贵的操作，LevelDB 定义了三个条件来决定是否需要做 compaction：
- Level 0 （最低层）的文件数量超过了限制；
- 其他层的数据量超过了限制；
- 一个文件的无效读取次数超过了限制。

## 结语

LevelDB 的代码量并不算很大，但是确实设计的很好。早就听说这是一个学习 C++ 的好材料，如今一读，确实如此。特别是对于我这种常年打 C 补丁的选手，LevelDB 实打实的给上了一节 OOP 的实践课。也推荐给所有想了解 LSM-Tree，学习 C++ OOP 范式的同学阅读。
