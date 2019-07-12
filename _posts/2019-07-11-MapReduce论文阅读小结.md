---
layout: post
title: "MapReduce论文阅读小结"
comments: true
description: "MapReduce论文阅读小结"
keywords: "分布式, 论文"
---

MapReduce 是 Google 在 2005 年提出的一种编程模型，它能以集群的方式处理大量的数据。MapReduce 是由 Map 和 Reduce 两个过程组成，这也是其名字的由来。

下面这张图参考了论文中的 MapReduce 的执行流程图:

![](http://ww1.sinaimg.cn/large/c9caade4gy1g4wx7e0stqj21ha0ogjwo.jpg)

首先是图中的 Master 和 Worker，它们分别 fork 自同一个用户程序。Master 和 Woker 在集群中扮演的角色就像它们的名字一样，Master 负责指派 task 给 Worker，Worker 则负责完成指派的 task。task 在集群中分为两类，分别是 map task 和 reduce task。上图中将输入文件分为 M = 6 份，即有 6 个 map task。这些 map task 经过 Worker 处理后存放到自己到本地磁盘上。在本地磁盘上被分为 R = 2 份，即有 2 个 reduce task。

map/reduce task 由 map/reduce funtion 来处理。这两种函数是由用户自定义的。map function 将 map task 的处理结果存放在 Worker 的内存缓冲区中，然后周期性的将缓冲区的内容写进磁盘，由 partitioning function 将磁盘上的数据分为 R 份。这时，正在执行 reduce function 的 Worker 通过网络远程读取磁盘上的数据，读取后的数据经过 reduce function 处理，写入文件中(通常是GFS管理的文件)。



GFS: 

- We conserve network band- width by taking advantage of the fact that the input data (managed by GFS [8]) is stored on the local disks of the machines that make up our cluster. GFS divides each file into 64 MB blocks, and stores several copies of each block (typically 3 copies) on different machines.
- input stored in GFS, 3 copies of each Map input file. all computers run both GFS and MR workers
- Reduce workers write final output to GFS (one file per Reduce task)
- Map input is read from GFS replica on local disk, not over network.
- Map worker writes to local disk, not GFS.
- master re-runs, spreads tasks over other GFS replicas of input.
- GFS has atomic rename that prevents output from being visible until complete.