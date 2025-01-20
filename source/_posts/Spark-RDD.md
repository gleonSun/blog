---
layout: title
title: Spark RDD
date: 2021-04-02 16:44:33
tags: Spark
categories: [分布式系统, 分布式计算, Spark]
---

# Spark RDD

弹性分布式数据集 （ Resilient Distrbuted Dataset），本质是一种分布式的内存抽象，表示一个只读的数据分区（Partition）集合。

RDD 本身是不存储数据的，且只有在调用例如 collect 时才真正执行逻辑。RDD 是不可变的，只能产生新的 RDD，其内部封装了计算逻辑。

## 弹性

- 在内存和磁盘间存储方式的自动切换，数据优先在内存缓冲，达到阈值持久化到磁盘
- 基于血缘关系（Lineage）的容错机制，只需要重新计算丢失的分区数据

## 特性

- `A list of partitions`

    分区列表。RDD 包含多个 partition，每个 partition 由一个 Task 处理，可以在创建 RDD 时指定分片个数。

- `A function for computing each split`

    每个分区都有个计算函数。以分片为单位并行计算。
    
- `A list of dependencies on other RDDs`

    依赖于其他 RDD 的列表。RDD 每次转换都会生成新的 RDD，形成前后的依赖关系，分为窄依赖和宽依赖，当有分区数据丢失时，Spark 会通过依赖关系重新计算，从而计算出丢失的数据，而不是对 RDD 所有分区重新计算。
    
- `Optionally, a Partitioner for key-value RDDs`

    K-V 类型的 RDD 分区器。

- `Optionally, a list of preferred locations to compute each split on`

    每个分区的优先位置列表。该列表会存储每个 partition 的优先位置，移动代码而非移动数据，将任务调度到数据文件所在的具体位置以提高处理速度。

## 转换

RDD 计算的时候通过 compute 函数得到每个分区的数据，若 RDD 是通过已有的文件系统构建的，则读取指定文件系统中的数据；若 RDD 是通过其他 RDD 转换的，则执行转换逻辑，将其他 RDD 数据进行转换。其操作算子主要包括两类：

- `transformation`，转换 RDD，构建依赖关系

- `action`，触发 RDD 计算，得到计算结果或将 RDD 保存到文件系统中，例：show、count、collect、saveAsTextFile等

RDD 是惰性的，只有在 action 阶段才会真正执行 RDD 计算。

## 任务执行及划分

- 基于 RDD 的计算任务

    从物理存储（如HDFS）中加载数据，将数据传入由一组确定性操作构成的有向无环图（DAG），然后写回去。

- 任务执行关系

    - 文件根据 InputFormat 被划分为若干个 InputSplit，InputSplit 与 Task 一一对应

    - 每个 Task 执行的结果来自于 RDD 的一个 partition

    - 每个 Executor 由若干 core（虚拟的，非物理 CPU 核） 组成，每个 Executor 的 core 一次只能执行一个 Task

    - Task 执行的并发度 = Executor 数 * 每个 Executor 核数

> - RDD 中用到的对象都必须是可序列化的，代码和引用对象会序列化后复制到多台机器的 RDD 上，否则会引发序列化方面的异常，可继承 Serializable 或使用 Kryo 序列化
> - RDD 不支持嵌套，会导致空指针

## 依赖关系

新的 RDD 包含了如何从其他 RDD 衍生所必需的信息，这些信息构成了 RDD 之间的依赖关系。

- 窄依赖
    
    每个父 RDD 的一个 partition 最多被子 RDD 的一个 partition 使用，例如：map、filter、union等，是一对一或多对一的关系。转换操作可以通过类似管道（pipeline）的方式执行。
   
- 宽依赖

    - 一个父 RDD 的 partition 同时被多个子 RDD 的 partition 使用，例如：groupByKey、reduceByKey、sortByKey等，是一对多的关系。数据需要在不同节点之间进行 shuffle 传输。

    - 遇到一个宽依赖划分一个 stage

## 自定义 RDD

继承 RDD 并实现以下函数，一般来说前三个比较重要。

- compute

    对 RDD 的分区进行计算，收集每个分区的结果

- getPartitions

    自定义分区器，获取当前 RDD 的所有分区

- getPreferredLocations

    本地化计算，调度任务至最近节点以提高计算效率