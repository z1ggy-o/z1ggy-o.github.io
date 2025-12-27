---
title: "2PC v.s. 3PC: 一句话的总结"
# date: 2022-07-31 12:15:08
# categories: 
#   - articles
tags: 
  - distributed-systems
toc: false
# isCJKLanguage: true
draft: false
# permalink: /pages/a32d83/
sidebar: auto
author: [Me]
---

最近在准备面试的过程中看到有这样一个问题，就是让比较一下 2PC 和 3PC。在网上找了一些文章来读，感觉都没有十分简洁地说明两者之间最基本的区别点。所以，在这里写一篇小文，表达一下自己对 2PC，3PC 最核心区别的理解。

最重要的最先说，在我看来两者的核心区别在于：**参与者间是否对 transaction commit/abort 建立了共识**。3PC 的参与者之间对 commit 的成立是具有共识的，2PC 则没有。

## 3PC 相比 2PC 带来了什么？

大家都知道，3PC 相比于 2PC 来说多了两个东西：
1. 增加了 time out
2. commit phase 被分割为了 prepare commit 和 do commit 两个部分

增加一个 prepare commit phase 带来了什么呢？是集群对 commit 这一决定的共识。

在 2PC 协议中，协调者单方面向参与者发送一次 commit 消息。这个消息有两个含义：
1. 让参与者进行 commit
2. 所有参与者都认可此 commit

但需要注意的是，对于一个 transaction，是 commit 还是 abort 这个决定本身只有协调者知道。一旦协调者故障，这部分信息就消失了。所以我们说 2PC 的协调者是单点故障点。

3PC 协议中，prepare commit 消息让一个 transaction 该 commit 还是 abort 这个决定本身被传导到了所有参与者处。如此一来，如果协调者故障，参与者们可以根据 prepare commit 的情况继续工作。

举例来说，time out 之后，我们从参与者中选出一个新的协调者。该协调者向其他参与者询问对于当前未完成 transaction 的信息。如果所有参与者都表示已经收到了 prepare commit，则证明我们可以对此 transaction 进行 commit；如果有部分参与者没有收到 prepare commit，我们可以选择 abort 或者再次尝试 commit 流程；如果谁都没有收到，则 rollback。

## 2PC 的参与者为什么不能接替工作？

读完上面的叙述中，大家可能会有疑问。明明 2PC 中的 commit 信息已经携带了所有参与者都同意 commit 的隐藏信息，为什么不能像 3PC 一样让参与者成为新的协调者继续工作呢？为了解决这个疑问，我们需要考虑一个极端情况：协调者在发送部分 commit 信息后故障，收到 commit 信息的参与者在 commit 后也发生故障。

在这个情况下，如果剩余的参与者之一成为新的协调者，因为所有参与者都没有接收到 commit 消息，我们只能认定这个 transaction 需要被 abort。要是在此之后，之前已经 commit 的故障参与者恢复了，灾难就到来了 -- 集群内各个服务器的数据不一致。

基于以上原因，2PC 在协调者发生故障后，只能阻塞等待。因为参与者自己不能判断能否对进行着的 transaction 进行 commit 或 abort。

3PC 则没有这个问题。同样的情况，因为只有在确认所有的参与者都收到 prepare commit 之后才会实际进行 commit。即，在可能出现 abort 决定情况下，系统中谁都还没有对此 transaction 进行 commit。所以，当故障服务器恢复后，不会有 2PC 例子中的数据不一致问题。

## 3PC 的问题

敏锐的人可能发现了，上面对 3PC 参与者可以安全接替工作的表述并不是在所有的情况下都成立的。这是因为 3PC 是基于 reliable network 环境设计的。

具体来说，如果发生了 network partition，又恰好把接收到和未接收到 prepare commit 消息的服务器们分别划到了不同的 partition 中。在 time out 后，一边会选择 commit，而另一边会选择 abort。一旦 network 重新恢复，我们也同样面对数据不一致问题。

因为在实际的应用中，我们的网络都是不稳定的，加之 3PC 增加了一轮通信，所以很少在工程中使用。
