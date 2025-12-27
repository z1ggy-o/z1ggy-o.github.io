---
title: "SOSP '97 - The design of a practical system for fault-tolerant virtual machines"
categories: 
  - paper notes
tags: 
  - distributed-systems
toc: false
isCJKLanguage: false
date: 2022-05-23 09:00:39
draft: false
permalink: /pages/1ec315/
author: 
  name: gyzhu
  link: https://github.com/z1ggy-o
---

[^1]: D. J. Scales, M. Nelson, and G. Venkitachalam, “The design of a practical system for fault-tolerant virtual machines,” SIGOPS Oper. Syst. Rev., vol. 44, no. 4, pp. 30–39, Dec. 2010, doi: 10.1145/1899928.1899932.

## MOTIVATION [^1]

A common approach to implementing fault-tolerant servers is the primary/backup approach. One way of replicating the state on the backup server is to ship changes to all state of the primary (e.g., CPU, memory, and I/Os devices) to the backup. However this approach needs high network bandwidth, and also leads to significant performance issues.

## CONTRIBUTION

Shipping all the changes to the backup server asks for high network bandwidth. To reduce the demand of network, we can use the "state-machine approach", which models the servers as deterministic state machines that are kept in sync by starting them from the same initial state and ensuring that they receive the same input requests in the same order.

There are three challenges:
1. correctly capturing all the input and non-determinism
2. correctly applying the inputs and non-determinism to the backup
3. low performance degradation

To handle these challenges, they implemented a fault-tolerant virtual machines in VMware vSphere 4.0. This system reduces the performance of real applications by less than 10%, and needs less than 20 Mb/s network bandwidth.

## SOLUTION

- There is one primary VM and one backup VM on different hosts. All input goes to the primary. And the input is sent to the backup VM via logging channel (network).
- A VM has a broad set of inputs and some additional information is needed for non-deterministic operations. They use the *VMware deterministic replay* to records the inputs of a VM and all possible non-determinism associated with the VM execution in a stream of log entries written to a log file.
- They design a protocol to ensure the failover (i.e., switching to the backup) is transparent to the clients. The core idea is that before send an output to the external world, we must need to make sure the backup VM has received the log entry that produces the output.
- The failure detection is handled by UDP heartbeating messages and  logging traffic.
- To avoid "split-brain" situation, they store a flag in the shared storage, so VMs can know if there is any other running primary.

## LIMITATION

The limitation is that the implementation only works for uni-processor machines because recording and replaying the execution of a multi-processor VM can lead to significant performance issues (accessing to shared memory is non-deterministic operation).

## MAIN TAKEAWAY

It is helpful to distinguish between internal and external and internal events of the system. For an infrastructure, only external events can really affect other applications.
