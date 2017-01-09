---
title: MapReduce原理与实现（一）：基本原理以及单机版的Go语言实现
date: 2017-01-05 10:37:08
categories:
- MapReduce原理与实现
tags:
- MapReduce
- 分布式
- Mit 6.824
---

## Introduction

MapReduce是用来处理大数据的一个编程模型。

用户一般规定了*map*方法来处理一个k/v对，而*reduce*方法来合并相关的中间值。

## Programming Model

计算任务以k/v对集合作为*输入*，并产生另外的k/v集合作为*输出*。用户通过*Map*和*Reduce*方法来控制计算。


本系列将实现一个MapReduce库。既可以学习Go语言编程，也学习分布式系统中的故障容错。

