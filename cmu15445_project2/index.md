# CMU 15-445 2021 Fall Project#2


> Because the course asks us to not sharing source code, here, I will only jot down some hits to help you (or maybe only me, kk) to finish the project. I will not even describe the process of any specific function, because I don't think that would be very different from public the source code.

## Task #1 - Page Layouts
Because we want to persist the hash table instead of rebuild it everytime, we need to design the layout that we use to store the hash table in the disks.

We have implemented the `BufferPoolManager` in previous project. Buffer pool itself only allocate page frame for us to use. However, which kind of `Page` is stored in the page frame? In this task, we need to create two kinds of `Page`s for our hash table:
- Hash table directory page
- Hash table bucket page

Since we are using the previous allocated memory space (we cast the `data_` field of `Page` to directory page or bucket page), understanding of memory management and how C/C++ pointer operation works are necessary.

For bitwise operations, because the bitmap is based on char arraies, we can only do bitwise operations char by char (at least, this is what I find).

### Hash Table Directory Page
This kind of page stores metadata for the hash table. The most important part is the **bucket address table**.

At this stage, only implement the necessary functions, and they are very simple. We will come back to add more functions in the following tasks.

### Hash Bucket Page
Stores key-value pairs, with some metadata that helps track space usages. Here, we only support fixed-length KV pairs.

There are two bitmaps that we used to indicate if a slot contains valid KV:
- `readable_`
- `occupied_`

`occupied_` actually has no specially meaning in enxtendible hashing because we do not need tombstone. However, we still need to maintain it, because there are some related test cases. I assume the teaching team still leave this part here so that they do not need to modify the test cases.

The page layout itself is very straightforward. Only the bitwise operations are a little annoying.

## Task #2 - Hash Table Implementation
This task is very interesting because we use the buffer pool manager that we implemented previously to handle the storage part for thus. We only need to use these APIs to allocate pages and store the hash table in these pages.

I recommand to implement the search at the very beginning then other funtions, since other operations also need search to check if a specific KV pair is existed.

### Search
For a given `key`, we use the given hash function to get the hash value, then through the directory (bucket address table) to get the corresponding bucket page. After that, we do a sequential search to find the key. 

Because we support duplicated keys, we need to find all the KV pairs that has the given key. Do not stop at the first matched pair.

### Insert
The core conponents of insertion part is split. Highly recommand to use pen and paper to figure how bucket split works before coding.

The insertion procedure is as follows:
1. Find the right bucket
2. If there is room in the bucket, insert the KV pair
3. If there is no room -> split the bucket

How to split one bucket? Assume we call the split target _split bucket_ and the newly created bucket _image bucket_.

- If $ global\\_depth == local\\_depth $:
    - Increase the `global_depth` by 1, so double the table size
    - The following steps are same as situation $ global\\_depth > local\\_depth $

- If $global\\_depth > local\\_depth$: 
    - Allocate a new page for the image bucket
    - Adjust the entries in the bucket address table
        - leave the half of the entries pointing to the split bucket
        - set all the remaining entries to point to the image bucket
        - also increase the `local_depth` by 1 because we need one more bit to separate them
    - Rehash KV pairs in the split bucket 
    - Reattemp the insertion
        - Should use the `Insert()` function because we may need more splits

Add your own test cases. The given test case is so small and cannot cover all situations.

### Remove
Remove itself is very simple. The complicated part is the merge procedure. The good thing is that, after finished split, the logic of merge became clear to us.

The project describtion gives a fairly thorough instructions for merge. Follow the instruction is enough.

Shrinking the table is very straightford since we use LSB to assign KV pairs to different buckets.

## Task #3 Concurrency Control

Try coarse-grained latch (lock) first, then reduce the latch range.

No special comments for this. You can do it!

## Result
Because the clang-tidy issue of Gradescope, I cannot test my code yet.
However, I tried some local test cases that covers splits (two types) and small buffer pools (to enforce page swap). It seems the code works well in the basic one thread situation.
