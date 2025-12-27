---
title: CMU 15-445 2021 Fall Project#4
categories: 
  - Projects
tags: 
  - DBMS
toc: false
isCJKLanguage: false
date: 2022-05-09 14:54:38
draft: false
permalink: /pages/056b0c/
sidebar: auto
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

We are implementing a lock-based concurrency control scheme in this project. More specifically, the **strict two phase locking protocol**.

The concept is not that hard. However, there are many implementation details we need to care about. Especially, sometimes we need to guess what is the test code wants.

You need to read the source code carefully. The instruction of this project is kind vague, or even wrong.

## Task 1 - Lock Manager

Read `transaction.h`, `transaction_manager.h`, and `log_manager.h` to learn the APIs first.

The log manager only communicates with transactions and the transaction manager. As the textbook says, the log manager tracks all the lock requests for different tuples; a transaction tracks all the locks it holds for different tuples.

All the works are around the management of the hash table in the `log_manager` and the two sets that track locks in `transaction`. We only need to implement the basic logic structure of each API in this task because we will modify them in the following tasks.

## Task 2 - Deadlock Prevention

We use **wound-wait** here, which means, when requesting a lock on a data item, the "younger" transactions wait for the "older" transactions, while the "older" transactions kill the "younger" transactions.

Because the `log_manager` and `transaction` track the lock requests for each tuple and each transaction, this part of work is

## Task 3 Concurrency Control

The instruction and code are inconsistent. The instruction asks us to maintain the write sets in transactions. However, the table write sets have already been handled by the APIs in `table_heap.cpp`.  As a result, actually, we do not need to maintain the tuple write sets by ourself.

Each transaction can execute several queries. We should consider this and modify our lock manager. For example, what should we do if a transaction inserts some tuples, then read them?

To achieve different isolation level with strict 2PL protocol, we need to use these locks properly in different executor. According to the lecture slides, we should:

+ **Serializable**: Obtain all locks first; ;plus index locks, plus strict 2PL.
+ **Repeatable Reads**: Same as above, but no index locks.
+ **Read Committed**: Same as above, but share locks are released immediately.
+ **Read Uncommitted**: Same as above but allows dirty reads (no share locks).

Some exceptions that are thrown from the lock manager cannot be fetched by the test code. So, letting the caller of the lock manager to handle lock fails is a better choice. This costed me few hours to debug..

## Result
![image.png](https://raw.githubusercontent.com/z1ggy-o/static_resources/main/img/cmu15445-project03-grades.png)
