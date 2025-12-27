---
title: CMU 15-445 2021 Fall Project#3
categories: 
  - Projects
tags: 
  - DBMS
toc: false
isCJKLanguage: false
date: 2022-04-30 12:37:56
draft: false
permalink: /pages/99d9e5/
sidebar: auto
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

In this project, we will Implement **executors** for taking query plan nodes and executing them.

We are using the iterator query processing model (i.e., the Volcano model). Each executor implements a loop that continues calling `Next` on its children to retrieve tuples and process them one-by-one.

## How `executor` works
Before start coding, we need to learn a lot from the related source code first. The instruction does not show all the details and I believe they did this intentionally.

As discussed in the lecture, DBMSs will convert a SQL statement into a query plan, which is a tree consisted by operator nodes. The executors that we implement define how we process these operators.

Inside an operator, we get the expression of the operator. Thus, to implement an executor, we need to get the expression of that plan node, and evaluate these expression in the right way to get the result.

In this project, I recommend you to read the test code first before anything else. The test code tells us the structure of the code base and how they are combined together.

The *plan node* and *execution engine* are the most important components.
- `ExecutionEngine` is defined in `execution_engine.h` and there is only one single API -- `Execute()`. In this function, it calls `ExecuteFactory` to create executor object and return a smart pointer to the caller.
- There is a `SetUp()` function in `executor_test_util.h` that does all the initializations for us. That's why we can use `GetExecutionEngine()` and `GetExecutorContext()` in the test file directly. (I was very curious about this.)
- Different operators have their own plan node. The executor can fetch information it needs from the passed in plan node. Thus, do check all the members of the plan node.

## Sequential Scan
This task needs us to read a lot code to know how these components work. In short, what we need to do is read all the tuples from the `TableHeap`.

Once you know how to read a tuple from a given table, this task is almost done. However, do remember to use the output scheme to build the output tuple. Also, use the given predicate to filtrate out unsatisfied tuples.

## Insertion
In this part we need to do both:
  1. Insert new tuples into the table
  2. Insert new indexes for these new tuples

There can have several tuples to be inserted in one query, and each table can have several indexes (based on different index keys).

All the components that we need to insert tuples and indexes can be find through the `catalog` of the table. More specifically:
- We can get table information from the catalog. Through the table information, we can get the container of the table (`TableHeap`), which provides the APIs to modify the table.
- We can get index information from the catalog. Similarly, through the index information, we can get the container of the index (`Index`), which provides the APIs to modify the index.

Because all the APIs are provided by the code base or ourself (i.e., the underlying buffer pool management and extendible hash index), there is no much code we need to write for insertion.

## Update
Update is very similar with insertion. As described in the project instruction, the APIs that create updated tuples has been provided, so the tuple update part is very simple.

We need to consider when and how to update the indexes. A hint for this is that the index of attributes in the table index is the same in the table, which means we can know if the index keys are updated.

## Delete
Delete itself is very straightforward. Just delete the tuple and related indexes.

## Nested Loop Join
Need to read the test case code to learn how to build the joined tuples. More specifically, there are two `Expression`s that we need in this task, one is from the predicate, which is used for check if two tuples are matched; another is from the output schema columns, we need to use it to create the output tuples.

## Hash Join
Hash join is more complicated than nested loop join. The good news (?) is that, we assume the hash table can fit in the memory, so the basic hash join algorithm is enough.

The basic hash join algorithm has two phases: build and probe.
Before we check any tuple of the inner table, we need to build the hash table for the outer table first. This phase should be done at the beginning.

Then, for each tuple of the inner table, we can use the hash table to find the matched tuples.
We need to create a hash table by ourself. This is the most difficult part. As the instruction told us, we should check `SimpleAggregationHashTable` to learn how to create one for hash joining. You should also check `aggregation_plan.h`, which contains some components that `SimpleAggregationHashTable` uses. This part of work may need deeper knowledge about C++ than other tasks.

We also need to get the hash keys using the given expressions. (P.S. the instruction in the website may not be updated. There is no `GetLeftJoinKey()` and `GetRightJoinKey()` member functions in the code base that I am using.)

## Aggregation
Since the hash table is given by the code base, we only need to use these expressions that the plan node gives to us.
We only need to care about the `GROUP BY` and `HAVING` clauses. Aggregations are handled by the given code.

## Distinct
After we know how to build our own hash table through the exercise of hash join, this task becomes quite easy. We only need to create another hash table, and use it to get distinct tuples.

## Result
![image.png](https://raw.githubusercontent.com/z1ggy-o/static_resources/main/img/cmu15445-project03-grades.png)
At the time of my submission, I was ranked first on the leaderboard. ^^



