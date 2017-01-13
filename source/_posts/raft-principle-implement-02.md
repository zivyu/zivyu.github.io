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

<!-- more -->

### Election restriction

Raft使用一个简单的方法，它保证了之前term的所有被committed entries，在新当选的leader都存在。

Raft在投票的过程中就保证了一个candidate赢得选举的条件是这个candidate必须包含所有被committed entries。为了赢得选举，candidate必须与集群中的大多数联系，这意味着每个被committed entry至少在这些大多数中的一台中存在。如果candidate的log和集群中的大多数比一样新或比他们更新，那么就可以认为这个candidate hold所有被committed entries。这个限制是通过The RequestVote RPC来实现的：

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

到目前为止，我们都假设集群中的server是固定的。但是在实际生产中，我们偶尔会变更集群配置，比如说替换到宕机的server。最简单的方式就是停掉整个集群，然后更新配置文件，然后重启集群，但是这会造成集群在短时间内不可用。

想要安全的更新配置文件，必须保证在集群转换期间的同一个term内不可能会出现两个leader。如下图，把配置文件直接替换成新的是不安全的，可能会出现两个leader。

![](/images/raft_member_change_01.png)

为了确保安全，配置变更必须使用两阶段的方式。

在Raft中，集群首先转换到称为*joint consensus*的状态，一旦joint consensus被committed，系统就开始使用新的配置文件。

joint consensus将旧的和新的配置文件联系起来了：

- Log entries被复制到所有的集群中。包括新、旧配置文件。
- 来自任何一份配置总的server可能会是leader
- 对于选举和entry，都需要新和旧的多数派的同意。

集群配置使用特殊的log entries来存储，当leader收到一个将C old转换到C new的请求的时候，它就作为joint consensus来保存这个配置，并使用之前的机制来复制该log entry。一旦一个server将new configuration增加到它的log之后，后面所有的决策，它就使用该configuration。这意味着leader使用C old,new来决定对于C old,new的log entry是否被提交。如果leader宕机，一个新的leader要么在C old，要么在C old,new下被选择。任何情况下，C new都不可能做出单边决议。

一旦C old,new被提交，只有C old,new的server才能被选举为leader。对于leader来说，现在创建一个C new并复制给server是安全的。当C new被提交之后，	现在就可以安全的下线新的配置中不存在的机器。如下图，任何时间点，C new 和 C old都不可以同时做出单边决议。

![](/images/raft_member_change02.png)

如上图，leader首先创建C old,new ，被提交之后，再创建C new 然后把它提交到C new中的多数。在任意时间点，C old和 C new都不能同时独立的做出决议。

更新配置的时候，有三个主要的问题。

第一个问题是新的server可能没有初始化log entries。当他们加入集群的时候，需要花一段时间来追上最新的数据，在这段时间，它可能不会提交新的log entry。为了避免这个问题，Raft在配置变更前引入了一个新的阶段，新的server加入集群的时候，是作为一个non-voting成员，意思是leader仅仅是复制日志给他们，而不把他们作为大多数考虑。直到该server追上leader的log。

第二个问题是集群的leader可能并在新的配置中。在这种情况下，一旦它提交了C new，就主动下线。

第三个问题被移除的server可能会影响集群。这些被下线的机器不再受到心跳信息，然后他们开始了一轮选举。它们会发送RequestVote RPCs，并带上新的term，这会使leader转换到follower状态。一个新的leader会被选举出来，但是被移除的server又会time out，然后开始选举。为了解决这个问题，当这个server确信当前的leader存在的时候，server会忽视RequestVote RPCs。具体来说，如果一个server收到了当前leader的心跳之后的最小time out时间内收到了一个RequestVote RPC要求投票，它不会更新它的term，也不会进行投票。这不会影响正常的选举，因为每个server在选举之前至少等待最小 election timeout。