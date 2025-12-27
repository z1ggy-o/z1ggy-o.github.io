---
title: 幻读，到底是怎么一回事儿
categories: 
  - articles
tags: 
  - DBMS
toc: true
isCJKLanguage: true
date: 2022-05-12 07:11:54
draft: false
permalink: /pages/053390/
sidebar: auto
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

幻读 (phantom phenomenon) 是数据库使用中经常提到的一个问题。一个常见的面试问题就是：某个数据库，在某个 isolation level 下，会不会出现幻读，以及其解决方法。

网上有许多关于幻读的文章，但是在读完之后发现，大多数的说明都浮于表面，好像作者们自己也并没有弄清楚幻读的本质。在本文中，我想利用数据库的一些高层抽象概念，来阐述幻读的本质。虽然不涉及任何的具体实现，但相信你在了解到这些概念之后，可以很快地理解幻读，以及各种幻读的处理方法。

幻读以及其解决方法核心都可以浓缩在一句话中，即:

> The phantom phenomenon, where conflicts between a predicate read and an insert or update are not detected, can allow nonserializable executions to occurs.

其中的关键词为: *predicate read*, *insert or update*, 和 *nonserializable execution*. 我们将以这些关键词为突破口，来理解幻读问题。

## 并行可能性：Isolation 和 Conflict Serializability

ACID 是数据库事务的重要特性。其中的 I，即 isolation，指的是“多个事务同时运行的时候，它们之间是相互孤立的”。

上面对 isolation 的标准定义我个人并不喜欢，因为它太过于抽象。依照我个人的理解，更具体一点儿说，isolation 的目标是保证多个事务同时运行的时候，不会因 race condtion 而使得数据库的一致性(consistency) 被破坏。

那么如何实现 isolation 呢？其关键就是并行事务运行的 serializability.

### Schedules and Serial Schedules

当多个事务同时运行的时候，因为任务调度是由操作系统在负责，这些事务所包含的操作会交错运行。这些操作的一个实际运行顺序被称为一个 _schedule_。可见，对于一组同时运行的事务，它们可能出现的 schedules 是非常多的 (有 n 个事务的话，会有 $n!$ 种可能)。

依据我们对 race condition 的理解，对于同一个 data item，如果并行访问中有写操作参与，就会有 race condition 的出现。那么，对于一组并行事务来说，如果其中有两个或以上的事务对同一个 data item 进行访问，且其中包含写操作，我们就说这些访问操作之间是会有冲突的 (conflict)。我们需要对其进行运行顺序进行控制，否则，最终的结果是不可控的，可能会破坏数据库的一致性。

### Conflict Serializability

想要控制事务运行顺序，最简单的方式就是顺序运行。即，一个事务运行完成之后，才运行另一个事务。这种运行顺序，就是 *serial schedules*.

Serial schedules 虽然保证了数据库的一致性，但事务的并行也随之消失了。为了提高系统的性能，我们还是想要事务在不破坏一致性的前提下并行工作。如果一些 schedules，它能够支持并行，且能保证其运行结果和 serial schedules 的结果相同，我们就叫它 _serializable schedules_。

我们之前提到过，在一个 schedule 中分属于不同事务的两个连续操作，如果它们访问同一个 data item，且其中有一个是写操作，那么它们之间就是冲突的。即，在这个 scheule 中，它们两者的运行顺序是不能改变的。但，如果不冲突，这两个操作的顺序就可以交换。

通过交换一个 schedule 中不冲突的连续操作，我们可以得到新的 schedule。如果一个 schedule，通过不断的交换各个事务间不冲突的操作，能得到一个 serial schedule，我们就说这个 schedule 具有 *conflict serializability* ，或者说它是 *conflict serializable* 的。

无疑，conflict serializable 的事务运行顺序，就是我们想要的运行顺序，因为在允许事务并行运行的同时，它的运行结果和事务依次顺序运行时的结果一致。

需要追加说明的是，serializability 不止 conflict serializability 一种。例如，还有 view serializability。但因为实现的难度等原因，几乎所有的数据库都是使用 conflict serializability。

## 一致性弱化：Isolation Levels

Serializability 固然可以保证数据库在事务并行时的一致性，但因为它的限制较多，使得系统的整体并行性能受到了压制。

有些时候，我们为了性能，根据实际的应用场景，可以牺牲一些对一致性的要求。这就是 isolation levels 的意义所在。SQL 标准中给出了四个不同级别的 isolation levels：
- **Serializable**: 最高级别，保证之前提到的 serializable execution.
- **Repeatable read**: 一个事务只能读取到其他事务 commit 后的值。而且，如果一个事务会对同一个 data item 读取多次，那么该事务完成最后一个读取之前，其他事务不能更改该 data item。
- **Read committed**: 仅保证只能读取到其他事务 commit 后的值。
- **Read uncommitted**: 未 commit 的数据也能读取。

四个等级的主要差距在 read 上，因为它们都保证没有 dirty write。即，如果一个 data item 已经被一个事务修改了，在该事务 commit 或 abort 之前，其他事务不能修改该 data item。

## 一致性保障：Concurrency Control Protocols

数据库的并行控制 (concurrency control) 子系统的作用，就是保证在多个事务运行的时候，可能出现的运行顺序 (schedules) 能够符合所指定的 isolation level 的要求。

Concurrency control protocols 是一些并行控制规则，依据这些规则工作，我们就可以控制可能出现的并行事务运行顺序。常见的规则类型有 *lock-based protocol*, *timestamp-based protocols*, 以及 _validation-based protocol_。

- Lock-based protocol 是通过给想要访问的 data item 上锁的方式来实现并行控制。这是一个比较通用的控制方式，在数据库以外的领域也被大量使用。
- Timestamp-based protocol 则是通过记录每个 data item 的读、写 timestamp ，并以之与事务的 timestamp 进行比较的方式来进行并行控制。
- Validation-based protocol 算是 timestamp-based protocol 的一个扩展。

在这里，我不对这些 protocol 的具体内容进行阐述。我们只需要知道一点，这些 protocol 的工作原理，不论是上锁，还是添加 timestamp，都有一个前提，就是**目标 data item 是已存在的**。这个隐性的前提看起来理所当然，但它就是幻读问题的根本所在。

## 终于，幻读：Read repeatable level 下，会有幻读吗？

### 幻读 Phantom phenomenon

幻读现象指的是，在一个事务中，多次运行同一个 query 所得到的结果不同。而这个结果，是其他事务添加或删除被读取的 tuple 而发生的。

幻读在 read uncommitted，read committed，以及 read repeatable level 下都会出现。但是 read uncommitted 和 read committed 本身还有 dirty read 和 unrepeatable read 的现象。为了避免混淆，我们以 read repeatable (RR) 下的情况来做例子，这也是许多面试题的假定条件。

要理解幻读出现的原因，先让我们再读一次文章开头的句子：

> The phantom phenomenon, where conflicts between a predicate read and an insert or update are not detected, can allow nonserializable executions to occurs.

可见，幻读的出现是有具体的场景的。第一，是在做 **predicate read** 的时候；第二，对于前面读取操作的目标 data item，有与其冲突的 **insert** 或 **update** 发生。

根据之前的内容我们已经知道，当多个并行事务对同一 data item 有相邻的读写访问时，这些读写访问操作之间是冲突的。这些冲突在 concurrency control protocols 的帮助下，是可以得到解决，使得数据库的一致性得到某种程度的保障。

既然读写是冲突的，我们也有 concurrency control 的帮忙，为什么幻读还会出现呢？答案，就在前面提到的各个 concurrency control protocols 的工作方式上。
我们以 lock-based protocol 中的 two-phase lock 为例来看看到底是怎么一回事：

- 当事务进行读取请求的时候，该事务会对将要访问的 tuple 加上共享锁，并维持到事务结束 (RR 条件下)。此时，如果有另外的事务想要修改被上锁的 tuple，需要获得该 tuple 的排他锁。所以，修改无法发生。没有问题。
- 但是，如果是不满足 predicate read 的限制范围的 tuple 被 update，因为该 tuple 没有被上锁，这个请求可以立即执行。假如 update 后的 tuple 满足了 predicate read 的限制条件。当我们再次做相同的读取请求时，就会有新的 tuple 被读取。
- 同理，insert 请求也可以直接运行。如果其他的事务新添加了满足限制范围的 tuple，那么再次运行相同的读取请求时，也会发现读取结果发生了变化。

可见，幻读出现的原因在于， 两个事务实际上有冲突，但**因为冲突的对象是还不存在，concurrency control 无法对其访问进行限制**，所以 unserializable schedules 得以出现。

### 避免幻读的方法

幻读发生的原因是我们无法探知到发生在还未存在的数据上的请求冲突。那么，解决此问题的核心就是，**将请求冲突转移到已存在的数据上**。

例如，可以将冲突转移到 table 的 metadata 上，或者转移到 index 上。即，将 table metadata 或 index 纳入 concurrency control 的管理范围。Serializable level 下不会有幻读问题，因为它同时只让一个事务访问 table，也就是将冲突转移到 table metadata。

将冲突转移到 table metadata 的方式会大大降低并行度，我们大都不会采用。转移到 index 上是一个不错的选择， *index-locking* 就是其中著名的方式之一。

根据 index concurrency control 的方式不同，我们可以有不同的方式来避免幻读。但其核心离不开“将请求冲突转移到已存在的数据上”，以保证只有 serializable schedules 能够出现。

## 总结

- 数据库通过一定的 concurrency control protocols 来保证多个事务并行运行时结果的正确性。
- 幻读的发生，是因为基本的 concurrency control protocols 不能探知到发生在还未存在的数据对象上的访问冲突。
- 解决幻读的方法是，将对未存在的数据对象上的访问冲突，转移到已存在的数据上去。著名的方法有 index-locking，next-key locking 等。
