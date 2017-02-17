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

## Interface

GFS提供了类似于文件系统的接口，文件由路径名来标识。

GFS就提供了一些关于文件操作的接口，比如说create、delete、open、close、read和write。此外，GFS还有*snapshot*和*record append*操作。

## Architecture

Gfs集群由一个*master*和多个*chunkservers*组成。文件被划分成固定的大小，叫做*chunks*。当chunk被创建的时候，master会给该chunk分配一个64位全局唯一的*chunk handle*，用来标识该chunk。chunk存储在Chunkserver中，通过chunk handle和字节范围可以读取制定的chunk。chunk默认有三个副本，但是用户可以指定副本数。

master包含了所有的文件信息。包括文件到chunk的映射，当前chunks的位置等等信息。

master和chunkserver之间保持心跳，这样maseter可以从心跳信息中得到chunkserver的状态。

## Single Master

使用单一的master极大的简化了系统的复杂度。

Client永远不会向master读取和写入文件，它只是向master得到它要联系的chunkserver，并将这些信息缓存一段时间，后续的操作就可以直接与chunkserver进行交互。

![](/images/gfs_01.png)

如上图，是一个客户端读取文件的步骤：

1. 客户端根据文件名和字节偏移得到一个chunk索引。
2. 客户端将文件名和索引发送给master。
3. master将对应的chunk handle和位置回复给客户端，客户端收到回复之后将会缓存这些信息。
4. 客户端然后发送请求给到其中一个副本。请求包含了chunk handle和字节范围。

## Chunk Size

Chunk的大小是非常关键的一个参数。在gfs中，chunk的大小是64MB。每个chunk都是chunkserver中一个普通的Linux文件。

设置一个比较大的chunk size有几个优点：
1. 减少了客户端与master之间的交互次数。因为对同一个chunk的读取，只需要初次得到该chunk的位置信息即可。
2. 因为chunk的比较大，所以客户端可能会对同一个chunk执行多个操作。所以可以与这个chunkserver保持长连接。
3. 减少存储在master中的metadata大小。

## Metadata

master存储三种类型的元数据：file和chunk的命令空间，file到chunk的映射关系，每个副本的位置。

### In-Memory Data Structures

因为metadata存储在内存中，所以master的操作是非常快的。

但是有一点可能需要担心，那就是chunk的数量和此后系统的整个容量都受限于mater的内存大小。这一点无需担心，因为对于每个64M大小的chunk，metadata的大小都没超过64KB。

### Chunk Locations

master并没有持久化chunkserver中chunk的信息，它是在启动的时候主动去chunkserver中拉取信息，并在随后的心跳中去更新这些信息。

### Operation Log

操作日志包含了关键元数据的历史变更记录。它是GFS的中心，不仅仅是因为它是唯一持久化数据的地方，而且是事件发生顺序的逻辑时间线。所有的文件和chunk，在它们被创建的时候就被逻辑时间唯一，永远的标记起来。

## Consistency Model

GFS有一个宽松的一致性模型。

### Guarantees by GFS

文件命名空间的变化（比如说创建文件）是原子的，这个由master来保证。

## 系统交互

我们的设计目标是尽量减少与master节点的交互。

### Leases and Mutation Order

![](/images/gfs_02.png)

1. 对于一个给定的chunk，client向master节点询问哪个chunkserver持有当前的lease，和其他副本chunk的位置。(如果没有任何一个持有lease，master则会选择一个授予lease。)
2. master回复primary的id和其他副本的位置。client会缓存这些数据。如果primary不可达或回复它不在持有lease，client就会再次向master询问。
3. client会把数据推送到所有的副本。每个chunkserver会暂时把数据存储在内部的LRU buffer cache中。把数据流从控制流中解耦出来，可以提高性能。
4. 当所有的副本都收到数据之后，client向primary发送一个write请求。请求中会带上之前push到副本中的数据的id。primary会给收到的变化分片连续的序列号，并将这些变化按照序号应用于本地状态。
5. primary将请求转发给所有的副本。每个副本按照相同的顺序应用变化。
6. 当副本都回复primary的时候，表明他们他们已经完成了操作。
7. primary然后回复客户端。如果有副本失败，primary也会回复给客户端。

### Data Flow

### Atomic Record Appends

## MASTER OPERATION