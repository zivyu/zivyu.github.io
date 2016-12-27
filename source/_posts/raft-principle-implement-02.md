---
title: Raft原理与实现（二）：安全性与成员变更
date: 2016-12-26 12:02:33
categories:
- Raft原理与实现
tags:
- Raft 
- 分布式
---

## Safety

之前的章节描述了Raft的leader选举和日志复制。但是如果按照之前的方式来，会有一些问题。比如说follower当选为leader之后可能会重写已经被commit的log，这样就会导致不同server的状态机执行了不同的command。

### Election restriction

Raft使用一个简单的方法，它保证了之前term的所有被committed entries，在新当选的leader都存在。

Raft在投票的过程中就保证了一个candidate赢得选举的条件是这个candidate必须包含所有被committed entries。为了赢得选举，candidate必须与集群中的大多数联系，这意味着每个被committed entry至少在这些大多数中的一台中存在。如果candidate的log和集群中的大多数比一样新或比他们更新，那么就可以认为这个candidate hold所有被committed entries。这个限制是通过The
RequestVote RPC来实现的：

- RPC包含了candidate的log
- 如果voter发现自己的log比candidate的log 更新，就拒绝这个投票。

Raft通过比较log中最新的entries的index和term来决定哪个更新。如果最新的entries有不同的term，则更大的term是更新的，如果有相同的term，谁的log更多，谁就是更新的。

### Committing entries from previous terms

当entry被复制到集群中大多数的时候，leader就知道这个entry是committed。如果leader在committing一个entry的时候cash了，随后的leader将会尝试完成这个entry的复制。但是一个leader不能立即推断出一个之前的term的entry是否是committed，即使这个entry被保存在了集群中的大多数server中。

如下图就展示了这种情况：一个老的entry已经被保存在了大多数server中，但是仍然被后面的leader给重写了。

![](/images/raft_safy_r.png)

如上图：

- 在（a）中，S1当选为leader，然后在index=2复制了部分log entry。
- 在（b）中，S1宕机，S5获得了来自S3、S4和它自己的投票，然后当选为leader，并在index=2写入了一个不同的entry。
- 在（c）中，S5宕机，S1重启之后当选为leader，然后开始复制log entry。这时候，来自term=2的log已经被复制到了大多数机器中，但是它还没有被committed。
- 在（d）中，由于S1宕机，S5获得了来自 S2， S3和S4的投票，然后从term=3开始重写entry。
- 在（e）中，如果S1在宕机之前把当前term的一个entry复制到大多数server中，并且这个entry被提交，这时候S5就不可能赢得选举。在这个点，所有之前的entry也被提交。

为了消除上图中这种情况，Raft永远不会通过副本数来commit之前term的log entries。只有当前term的log entries才会通过计算副本数被commit。并且一旦当前term的entriy被committed，根据 Log Matching Property，所有之前的log entries也会被提交。

### Safety argument

通过之前的选举和日志复制，然后再加上上面的限制。完整的Raft算法已经描述完了。现在来证明 Leader Completeness Property。

现在假设leader在term T 提交了一个当前term的log entry，但是这个log entry在随后的term没有被leader保存。

假设term U是大于term T的最小的term，并且term U的leader没有包含这个log entry。

1. 被提交的entry在leader U选举的时候肯定没有被保存在log中。因为leader不会删除或覆盖自己的log。
2. leader T把这个entry复制到集群中的大多数，并且leader U收到了来自集群中大多数server的投票。因此，至少有一个server（voter）收到了来自leader T的entry，并且给leader U投了票。像下图所示，这个server就是一个矛盾点。
3. voter必须在给leader U投票之前收到了被leader T提交的entry。否则它会拒绝来自leader T的AppendEntries请求，因为它当前的term比T的高。
4. 当voter投票给leader U的时候，它仍然保存着这个entry。因为每个中间的leader都包含了这个entry，leader永远不会移除entries，并且follower只会移除和leader冲突的entries。
5. voter将票投给了leader U，所以leader U的log必须和voter至少是一样新的。这就导致了两个矛盾。
6. 首先，如果voter和leader U有相同的最新的entry，leader U的log要至少和voter的log一样长，所以它的log包含了voter的log的每个entry。这就是一个矛盾。
7. leader U 最新的log entry的term必须大于voter的log entry。此外，它会大于T，因为voter的最新的log至少是T。
8. 因此，假设不成立。

大于T的leader的所有term必须包含了所有在term T被提交的entries。

![](/images/raft_safy_02.png)

如上图，如果S1在当前term提交了一个新的log entry，对于随后的term U，S5被选为leader，这时候必须有一个S3接受了这个log entry，并且给S5投票。

## Cluster membership changes

到目前为止，我们认为