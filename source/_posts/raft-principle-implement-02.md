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

之前的章节描述了Raft的leader选举和日志复制。但是如果按照之前的方式来，会有一些问题。比如说

