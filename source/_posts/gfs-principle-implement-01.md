---
title: Gfs原理
date: 2017-01-16 14:34:08
categories:
- 分布式学习
tags:
- Gfs
- 分布式
- Mit 6.824
---

对于分布式的学习，在理解了MapReduce的原理，及实现基本的MapReduce框架之后，可以说是吃了一盘开胃菜。那么从本篇开始，就要开始我们的主食。我们选择从：[The Google File System](https://pdos.csail.mit.edu/6.824/papers/gfs.pdf) 开始。

为什么选择GFS作为开始呢？

因为在分布式学习中，大部分的内容都在GFS中有所体现，包括性能、故障容错、一致性等。

<!-- more -->

## ABSTRACT


## Architecture

Gfs集群由一个单独的*master*和多个*chunkservers*组成。文件被划分成固定大小的*chunks*。当chunk被创建的时候，master会给该chunk分配一个64位全局唯一的*chunk handle*，用来标识chunk。chunk存储在Chunkserver中，默认有三个副本，但是用户可以指定副本数。

master包含了所有的文件信息。包括文件到具体chunk的映射，当前chunks的位置等等信息。master也会定期和chunkserver发送心跳信息。

## Single Master

Client永远不会向master读取和写入文件，它只是向master得到它要联系的chunkserver。

![](/images/gfs_01.png)

如上图，一个客户端读取文件的步骤：

1. 客户端根据文件名和字节偏移得到一个chunk索引。
2. 客户端将文件名和索引发送给master。
3. master将对应的chunk handle和副本位置回复给客户端，客户端收到回复之后将会缓存这些信息。
4. 客户端然后发送请求给到其中一个副本。请求包含了chunk handle和字节范围。

## Chunk Size