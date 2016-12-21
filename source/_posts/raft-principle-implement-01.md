---
title: Raft原理与实现（一）：简介、选举与日志复制
date: 2016-12-21 22:45:37
categories:
- Raft原理与实现
tags:
- Raft
---

## Raft是什么

Raft是来管理复制日志的一个一致性协议。它和（multi-）Paxos有一样的效果，但是它的结构却和Paxos有所不同。简单点来说，它比Paxos更容易理解，也更容易实现。相信读过Paxos的读者已经体会过Paxos的晦涩、难懂，以及难以实现之处。而Raft直接提出了实现该协议的几个要素：leader选举、日志复制和安全性，甚至给出了伪代码。相比Paxos，Raft通过加强对状态一致性的约束来减少需要考虑的状态数目。

## Introduction

一致性算法允许一组机器一起工作来应对某些机器宕机。在过去十来年，Paxos一直在这方面占主导地位：许多实现都是基于Paxos，或受Paxos的影响。但是，Paxos实在是太难懂了，此外，还需要一些复杂的改变来支持实际的应用。

在与Paxos做了一番斗争之后，我们提出了一个新的一致性算法，这个算法就叫Raft，并且它的首先目标就是要容易理解。

Raft有几个特性：

 * Strong leader：相比其他协议，Raft对leadership作出了更强的约束，比如说：日志只能从leader流向其他servers。
 * Leader election：Raft使用随机的时间来选举leader。
 * Membership changes：Raft使用了一个joint consensus方式来应对成员变更。这让整个集群在成员变更期间依然能提供服务。

## Replicated state machines

在分布式系统中，使用Replicated state machines来解决各种各样的故障容错问题。

Replicated state machines使用replicated log来实现，每个server存储一系列的commands，这些commands将会被状态机有序执行。每个log都会以相同的顺序存储相同的command。

一致性算法是的工作就是保证replicated log的一致性。

对于一个实际的系统，一致性算法有下列属性：

* safety：不会返回一个错误的结果。
* available： 大多数server可用的情况下，系统就是可用的。
* 不依赖时钟来确保log的一致性
* 当集群中的大多数机器能工作时，少数机器不会影响整个集群的性能。

如下图，是一个Replicated state machines的结构：

![](/images/raft_state_machine.png)




