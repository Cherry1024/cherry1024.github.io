---
layout:     post
title:      MapReduce: Simplified Data Processing on Large Clusters
subtitle:   数据领域经典programming model——MapReduce
date:       2020-10-15
author:     CY
header-img: img/post-bg-food3.jpg
catalog: 	 true
tags:
    - Big Data
    - MapReduce
---


## 简介

MapReduce是一个处理和生成超大数据集的编程模型和相关实现。基于MapReduce，用户只需通过Map和Reduce函数描述自己的计算问题，而不用关心计算在哪个机器上进行、相互之间如何通信、机器故障如何处理等复杂的问题。

> MapReduce is a programming model and an associated implementation for processing and generating large datasets that is amenable to a broad variety of real-world tasks. Users specify the computation in terms of a map and a reduce function, and the underlying runtime system automatically parallelizes the computation across large-scale clusters of machines, handles machine failures, and schedules inter-machine communication to make efficient use of the network and disks. 

### What is MapReduce

MapReduce 是 Google设计的一种用于大规模数据集的分布式模型，它具有支持并行计算、容错、易使用等特点。它的设计目标如下：

- 支持并行
- 用于分布式
- 能够进行错误处理（比如机器崩溃）
- 易于使用（程序员友好）
- 负载均衡

### Execution Overview

MapReduce 模型主要分为 2 个部分：**Map** 和 **Reduce**（都是用户自己写）。

在 Map 过程中，Map 函数会获取输入的数据，产生一个临时中间值，它是一个 Key/Value 对，然后MapReduce Library 会按 Key 值给键值对（K/V）分组然后传递给 Reduce 函数。而后，Reduce 接收到了这些 K/V 对，会将它们合并。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gjqefnm8whj310b0u0k02.jpg" alt="image-20201015144338374" style="zoom: 25%;" />

1. 由用户程序中调用的 MapReduce Library 将文件分成 M 块（M 要远大于 Map Worker 的数量，每块大小16MB~64MB），然后用户程序在一组机器集群上创建大量的程序副本，fork出多个子进程。此时，进入 MapReduce 过程
2. 程序副本中有一个Master程序和多个Worker程序，Master程序负责给空闲的Worker分配任务（M个Map任务和R个Reduce任务，M+R >> Machines.num）
3. 被分配了Map任务的Worker程序读取相应的输入数据片段，从数据片段中解析出kv对，然后把kv对传给Map函数，Map函数产生intermediate key/value对并缓存在内存中。
4. Map Reducer缓存中的kv对通过分区函数分成R个分片，周期性写入本地磁盘，根据key的不同分别写入R个本地文件中。这些kv对在本地磁盘上的存储位置将被回传给Master，由Master负责把这些存储位置再传送给Reduce Worker。**Map(k1,v1) -> list(k2,v2)**
5. （Map过程完成后）Reduce Worker接受到Master发来的存储位置后，使用RPC从Map Worker所在主机的磁盘上读取这些数据。当Reducer Worker程序读取了所有intermediate key/value pairs后，通过对key进行排序使得具有相同key值的数据聚合在一起。（如果内存不够，可以使用外部排序）
6. Reduce Worker程序遍历排序后的中间数据，对于每一个唯一的中间key值，Reduce Worker程序将这个key值和它相关的中间value值集合`<key,value[num]>`传递给用户自定义的Reduce函数。Reduce函数的输出被追加到所属分区的输出文件。**Reduce(k2,list(v2)) -> list(k3,v3)**
7. 当所有的Map和Reduce任务都完成之后，Master唤醒用户程序。此时，用户程序里对MapReduce调用才返回。

成功完成任务后，MapReduce的输出存放在R个输出文件中，一般不做合并，而这些文件往往又被作为另外一个MapReduce的输入

![image-20201015150738644](https://tva1.sinaimg.cn/large/007S8ZIlly1gjqefsyjoyj311a0jwdo1.jpg)

![image-20201015181532591](https://tva1.sinaimg.cn/large/007S8ZIlly1gjqefw90oaj30ni0boq9d.jpg)

### Master 

Master会记录每个task的状态*(idle, in-progress, or completed)*，并记录不是idle状态的机器。Master是map worker 和 reduce worker的桥梁，每个map task完成后完成后会将其产生的R个文件的路径和大小传输给Master，然后Master则会将这些信息push给那些处于 *in progress*的reduce worker。

### Fault Tolerance

和大多数Unix工具一样，运行MapReduce作业通常不会修改输入，除了生成输出外没有任何副作用。当遇到崩溃和网络问题时，任务可以安全地重试，任何失败任务的输出都被丢弃。因此，容错的设计是十分方便的，MapReduce可以保证作业的最终输出与没有发生错误的情况相同，尽管这其中不得不重试各种任务。

🌲 MapReduce 中的 Master 会定期进行 checkpoint 备份，如果 Master 宕机，会根据之前的 checkpoint 进行恢复，但是恢复期间，MapReduce 任务会中断。

🌲 Master定期向worker发送HeartBeat来检测worker是否还在正常工作，若无应答或应答有误，则认定宕机。

​		🍃如果正在工作的map worker宕机了，那么这个task被初始化为idle,重新分配到正常的worker上，即使已经完成了一些工作，所有reduce worker会被告知，要去宕机的worker上读取的会去另外正常的worker获取。

> 因为map worker处理后的中间结果存在内存/local disk中，宕机后无法获取

​		🍃若reduce worker宕机，任务不需要重做，因为处理后的结果保存在全局存储中。

🌲 每个工作中的任务把它的输出写到私有的临时文件中，每个Reduce任务生成一个这样的文件，而每个Map任务则生成R个这样的文件：

- 当一个Map任务完成时，Map Worker发送一个包含R个临时文件名的完成消息给Master。如果Master从一个已经完成的Map任务再次接收到一个完成消息，Master将忽略这个消息，否则Master将这R个文件的名字记录在数据结构里。
- 当一个Reduce任务**完成时**，Reduce Worker以原子的方式把临时文件**重命名为最终的输出文件**。如果同一个Reduce任务在多台机器上执行，针对同一个最终的输出文件将有多个重命名操作执行，MapReduce依赖底层文件系统提供的重命名操作的原子性**atomic rename** 来保证最终的文件系统状态**仅仅包含一个Reduce任务产生的数据**。

### Load Balance, Backup Task and Locality

#### Load Balance

📚 一开始文件分块时，分为M块，远大于Map Worker的数量的就有助于负载均衡，还有个好处是当某个worker宕机时，可以将任务迅速分配开，分到多个worker上，如果M比较小，有可能当一个worker宕机时，它的任务不够分配到剩下的worker中，会有worker闲置（加速了 fault tolerance 后的 recovery）。

#### Backup Tasks

📚 **straggler**指的是某个 worker 执行 task 时间过长，导致其他已完成 task 的 worker 都在等待这个 worker 完成（因为任务之间有依赖性），这一执行时间过长的 worker 也被称为**straggler**。

​	🍃 文章针对这一问题的做法是**把每个 task 分得尽量小，即 M(map task 的数量) 和 R(reduce task 的数量) 的值要远远大于机器数量**。

​	🍃 除了 task 过大，出现 straggler 的原因也可能是机器本身的硬件问题，哪怕 task 已经分得很小了。文章解决这一问题的做法是MapReduce 会对未完成的任务（in-progress） 定时执行备份执行操作，即出现straggler时，把这些正在某些 Worker 上执行但未完成的任务再次分配给其他 Worker 去执行，不论这个任务被哪个 Worker 完成都会被标记为已完成。这在文中称为 Backup Task。

#### Locality

网络带宽是一个相当匮乏的资源，所以将Worker调度到相应的输入数据所在的机器，从本地机器读取输入数据，从而减少网络带宽的消耗。这也是为什么我们所设的M值最好使得每个独立任务都处理不超过64M的输入数据，因为在GFS中每一个Chunk的大小就是64M，如果输入数据大于64M，就可能得从别的远程机器上读取其它Chunk中的数据，也就加大了网络开销。

📚 在任意分布式系统中，当 worker 数量增多，网络通信的负载都会变大。文章利用了 MapReduce 和 GFS 架设在同一组机器上的特性，从而**让 Map 过程从 GFS 读取文件时尽可能读取处于本地磁盘的的 copy**（GFS 为每份数据创建了三份 copies），在本地磁盘找不到时才读取其他 worker 磁盘的数据，这样就大大减少了网络的开销，而文中又称这一特性为 Locality。

#### Combiner

**📚 如果在 Map 任务中有一个 key 特别多，可能会拖慢整个网络的速度，该怎么办？（例如，在字数统计的例子中，the 这个词的数量特别多）**

MapReduce 给用户提供了一个 Combiner 函数，这个函数可以将结果在发送到网络之前进行合并，例如发送键值对<”by”, 3>。

### Conclusion

**性能(performance)，一致性(consistency), 容错(fault tolerance) 是分布式系统中比较关注的问题。**

MapReduce主要是关注性能和容错：

🌲 为了提升性能，可以将task分得更加小以获得更好的负载均衡，通过`backup task`来降低 straggler 对整个系统的影响，通过` locality` 来减少网络的负载。

🌲 针对容错机制，则通过为 task 设定状态，失败的 worker 的 task 被重置为 idle 状态，从而找到新的 worker 重新执行这些 task。

每种框架都有其适用场景，而对于 MapReduce，首先就是**任务要能够表达成一个或多个 MapReduce 过程**，文中提到的一些任务包括: Distributed Grep、Distributed Sort、Count of URL Access Frequency、ReverseWeb-Link Graph、Inverted Index等；其次**数据量要足够大**才能显示出 MapReduce 的效率(其实对于任意分布式系统基本都是这样, 否则整个系统调度的开销比计算的开销还要大)；还有就是**涉及到多次 shuffle (也即是多个 MapReduce 过程)时，由于要与磁盘多次交互，因此虽然能够实现，但是效率很低，**这时候就要考虑其他更灵活的框架了。不要局限于一定要把算法表达成 MapReduce 过程，而是可以考虑更加灵活的框架，如 spark 等。

### 参考

- [MapReduce: Simplified Data Processing on Large Clusters](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)
- [分布式系统笔记(1)-MapReduce](http://wulc.me/2019/01/14/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E7%AC%94%E8%AE%B0(1)-MapReduce/)
- [MapReduce论文笔记](http://hecenjie.cn/2020/02/01/%E3%80%8AMapReduce%E3%80%8B%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0/)

