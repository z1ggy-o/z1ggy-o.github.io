# File Systems Unift as Distributed Storage Backends: Lessons from 10 Years of Ceph Evolution


## Short Summary {#short-summary}

This paper mostly consists of two parts. The first part tells us why the `FileStore` has performance issues.
And the second part tells us how Ceph team build `BlueStore` based on the
lessons that they learnt from `FileStore`.

The main ideas of `BlueStore` are:

1.  Avoid using local file system to store and represent Ceph objects
2.  Use KV-store to provide transaction mechanism instead of build it by ourself


## What's the problem {#what-s-the-problem}

There is a software called `storage backend` in Ceph. The `storage backend` is
responsible to accept requests from upper layer of Ceph and do the real I/O on
storage devices.

Ceph used to use commonly used local file systems based storage backend (i.e.,
FileStore). FileStore works, but not that well. There are mainly three
drawbacks in FileStore:

1.  Hard to implement efficient transactions on top of existing file systems
2.  The local file system' metadata performance is not great (e.g., enumerating
    directories, ordering in the return result)
3.  Hard to adopt emerging storage hardware that abandon the venrable block
    interface (e.g., Zone divecies)


## Why the problem is interesting {#why-the-problem-is-interesting}

1.  Because storage backends do the real I/O job, the performance storage
    backends domains the performance of the whole Ceph system.
2.  For years, the developers of Ceph have had a lot of troubles when using local
    file systems to build storage backends


## The Core Ideas {#the-core-ideas}


### The problems of FileStore {#the-problems-of-filestore}

-   **Transaction**: in Ceph, all writes are transactions. There are three options for providing transaction on top of local file systems: leveraging file system internal transaction; implementing the write-ahead logging (WAL) in user space; and using a key-value store as the WAL
    -   leveraging file system's transaction is not a good idea because they usually lack of some functionalities. And also, file system is in the kernel side, it is hard to expose the interfaces.
    -   User space WAL: has consistency problem (not atom operations, because it is
        a logical WAL, it use read/write system calls to write/read data to/from
        logging part). The cost for handle the problem is expensive. Also slow
        read-modify-write and double-write (logging all data) problems.
    -   Using KV-store: This is the cure. However, there is still some unnecessary
        file system overhead like _journaling of journal_ problem

-   **Slow metadata operations**: enumeration is slow. The read result from a object
    sets should in order, which file systems do not do. We need to do sorting
    after read. To reduce the sorting overhead, Ceph limits the number of files in
    a directory, which introduces _directory splition_. The dir splition has
    overhead.

-   **Does not support new storage hardware**: new storage devices may need
    modifications in the existing file systems. If Ceph uses local file systems,
    the Ceph team can only wait for the developers of the file systems to adopt
    the new storage.

In the paper, there are more details. In summary, the reasons of above problems are:

1.  File system overhead
2.  Ceph uses file system metadata to represent Ceph object metadata. (i.e.,
    object to file, object group to diectory) The file system metadata operations
    are not fast and also may have some consistent issues.


### BlueStore {#bluestore}

1.  Does not use local file system anymore. Instead, store Ceph objects into raw
    storage directly. This method avoids the overhead of file systems.
2.  Use KV-store (i.e. RocksDB) to provide transaction and manage Ceph metadata.
    This method provides much faster metadata operations and also avoid building
    transtion mechanism by Ceph developer.
3.  Because RocksDB runs on top of file systems. BlueStore has a very simply file
    system that only works for RocksDB called BlueFS. The BlueFS stores all the
    contents in logging space and cleans invalid data periodly.

If you understand the reason of why FileStore performs not well, you can simply
understand the choices they did when build BlueStore.

BlueStore still has some issues. For example, because BlueStore do not use file
systems, it cannot leverage the OS page cache and need to build the cache by
itself. However, build a effective cache is hard.

