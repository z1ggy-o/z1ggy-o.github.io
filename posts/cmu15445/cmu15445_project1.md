---
title: "CMU 15-445 2021 Fall Project#1"
date: 2022-03-14T21:23:00+08:00
permalink: /pages/928c89/
sidebar: auto
categories:
  - Projects
tags:
  - DBMS
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

> Because the course asks us to not sharing source code, here, I will only jot down some hits to help you (or maybe only me, kk) to finish the project. I will not even describe the process of any specific function, because I don't think that would be very different from public the source code.

## Task #1 - LRU Replacement Policy
`BufferPoolManger` contains all the frames.
`LRUReplacer` is an implementation of the `Replacer` and it helps `BufferPoolManger` to manage these frames.

This `LRU` policy is not very "LRU" in my opinion. Refer the test cases we can see, if we `Unpin` the same frame twice, we do not update the access time for this frame. Which means that we only need to inserte/delete page frames to/from the LRU container. There is no positon modification (assume we use conventional list + hash table implementation).

You may need to read the code associated with task 1 and task 2 before start programming for task 1. Thus, you can understand how `BufferPoolManager` utlizes the `LRUReplacer`.

Actually, it is the `BufferPoolMangerInstance` managing the pages in the buffer. The `LRUReplacer` itself only contains page frames that we can use for storing new pages.
In other words, the reference (pin) count of pages that existed in the frames that in the `LRUReplacer` is zero, and we can swap them out in anytime.

Since we need to handle the concurret access, latches (or locks) are necessary. If you are not fimilar with locks in C++ like me, you can check the `include/common/rwlatch.h` to learn how Bustub (i.e., the DBMS that we are implementing) uses them.

## Task #2 - Buffer Pool Manager Instance
We use `Page` as the container to manage the pages of our DB storage engine. `Page` objects are pre-allocated for each frame in the buffer pool. We reuse existed `Page` objects instead of creating a new one for every newly read in pages.

We <span class="underline">pin</span> a page when we want to use it, and we <span class="underline">unpin</span> a page when we do not need it anymore. Because we are using the page, the buffer pool manager will not move this page out. Thus, <span class="underline">pin</span> and <span class="underline">unpin</span> are hints to tell the pool manager which page it can swap out if there is no free space.

Caution, `frame` and `page` are refering to different concepts. `page` is a chunk of data that stored in our DBMS; `frame` is a slot in the page buffer that has the same size as the `page`. So, use `frame_id_t` and `page_id_t` at the right place.

The comments in the base code is not very clear. They use "page" to refer both data pages and page frames, which is confusing. Make sure which is the target that you want before coding.

`BufferPoolManager` uses four components to manage pages and frames:

-   `page_table_`: a map that stores the mapping relationship between `page_id` and `frame_id`.
-   `free_list_`: a linked-list that stores the free frames.
-   `replacer_`: a `LRUReplacer` that stores used frames with zero pin count.
-   `pages_`: stores pre-allocated `Page` objects.

In terms of concurrency control, because we only have one latch for a whole buffer, the coarse-grained lock is unavoidable.

`BufferPoolManager` is the `friend` of `Page`, so we can access the `private` members of `Page`. (This is a good example about when to use `friend` -- when we need to change some member variables but we do not want give setters so that every one can change them.)

If we can do three things right, this task is not that difficult:

-   Move page to/from LRU.
-   Know when to flush a page. (Read points are very clear).
-   Which page metadata we need to update.

**Critical hints:**

-   Do read the header file and make sure your return value fits the function description. (I wasted few hours just because I returned `false` in a function, however, they assume we should return `true`  in that case. Do not use your own judgement, just follow the description.)
-   What will happen if we `NewPage()` then `Unpin()` the same page immediately?
-   Do not use the iterator after you erased the corresponding element. The iterator is invalided, however, the compiler will not warn you.

## Task #3 - Parallel Buffer Pool Manager
Task 3 is very straightforward. If our `BufferPoolManagerInstance` is implemented in the right way, the only thing we need to do here is to allocate mutiple buffer instance and call corresponding member functions for each instance.

Some people have problems with the start index assignment and update. Please make sure you do everything right as the describtion told us.

## Result
Passed all test cases with full grades.

![Project#1 grades](https://raw.githubusercontent.com/z1ggy-o/static%5Fresources/main/img/202203142113296.png)
