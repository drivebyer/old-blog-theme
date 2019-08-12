---
layout: post
title: 'MapReduce论文阅读小结'
subtitle: ''
date: 2019-07-11
categories: 技术
cover: ''
tags: 分布式
---

MapReduce 是 Google 在 2005 年提出的一种编程模型，它能以集群的方式处理大量的数据。MapReduce 是由 Map 和 Reduce 两个过程组成，这也是其名字的由来。

下面这张图参考了论文中的 MapReduce 的执行流程图:

![](http://ww1.sinaimg.cn/large/c9caade4gy1g4wx7e0stqj21ha0ogjwo.jpg)

首先是图中的 Master 和 Worker，它们分别 fork 自同一个用户程序。Master 和 Woker 在集群中扮演的角色就像它们的名字一样，Master 负责指派 task 给 Worker，Worker 则负责完成指派的 task。task 在集群中分为两类，分别是 map task 和 reduce task。上图中将输入文件分为 M = 6 份，即有 6 个 map task。这些 map task 经过 Worker 处理后存放到自己到本地磁盘上。在本地磁盘上被分为 R = 2 份，即有 2 个 reduce task。

map/reduce task 由 map/reduce funtion 来处理。这两种函数是由用户自定义的。map function 将 map task 的处理结果存放在 Worker 的内存缓冲区中，然后周期性的将缓冲区的内容写进磁盘，由 partitioning function 将磁盘上的数据分为 R 份。这时，正在执行 reduce function 的 Worker 通过网络远程读取磁盘上的数据，读取后的数据经过 reduce function 处理，写入文件中(通常是GFS管理的文件)。

既然 MapReduce 是为处理大量数据而设计，并且涉及到集群。那么在它内部必然有设计到独到之处，下面是 MapReduce 设计中的一些优点:

- reduce effect of slow network
- good load balance
- fault tolerance

#### 1、reduce effect of slow network

很多大型的系统设计，网络是一个很大的瓶颈。在 MapReduce 的设计中，执行 reduce task 的 Worker，需要从执行 map task 的 Worker 的本地磁盘上远程读取数据。这是唯一一处设计到数据的网络传输。其余地方，数据都是存储在本地。例如输入数据，存放在本地，由 GFS 统一管理。执行 map 后的数据依然是存放在本地的磁盘上(not GFS)。

#### 2、good load balance

在集群中，如果有一些机器的处理速度很快，它们就有可能早早的完成了任务空闲了下来。这样整个系统的完成时间就会变长。为了解决这个问题，通常将 M 设置得很大，一般要远大于集群中机器的数量。这样一来，通过 Master 的调度，性能高的机器会执行更多的 task。

#### 3、fault tolerance

在集群中 failure 分为两种情况:

- Worker Failure
- Master Failure

为了处理 Worker Failure，Master 会周期性的 ping 各个 Worker。如果在一段时间内没有收到某个 Worker 的 pong，Master 就会将它标记为 failure。所有在这个 Worker 上完成的 map task 都会重新执行(因为无法获取一个宕机节点上的数据)。Master 会将这些 map task 重新调度给其他的 Worker(这个reschedule操作要通知到集群中所有正在执行reduce task的Worker)。已经完成的 reduce task 不用重新执行，因为 reduce task 的数据存放在 GFS。

为了处理 Master Failure，Master 会周期性的将自己节点上的各种数据结构记录下来(checkpoint)，一旦发生错误要么重启计算，要么从上一次记录的地方恢复。

