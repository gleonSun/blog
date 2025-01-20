---
layout: title
title: HBase 概述
date: 2020-10-30 11:09:23
tags: HBase
categories: [分布式系统, 分布式存储, HBase]
---

# HBase 概述

从使用角度来看，HBase 包含了大量的关系型数据库的基本概念——表、行、列，我们通常也说 HBase 是面向列式的存储，包括在面试一些小伙伴的时候也聊到过这个问题，不少人都认为 HBase 是列存的，但这个理解是有偏差的。

我们先简单说一下行存和列存。行式存储数据库连续地存储整行，列式存储数据库以列为单位聚合数据，然后将列值顺序地存入磁盘，这种存储方法不同于列式存储的传统数据库。下图展示了列式存储和行式存储的不同物理结构：

<!--居左格式显示-->
<div align=center>
<img src="https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/bg-1737346418768.png" align="center" width = "400" height = "370" alt="RowOrColumn Storage"/>
</div>

<!-- 去除 Viewer does not support full SVG 1.1 水印-->
<!--sed -i 's/<text text-anchor="middle" font-size="10px" x="50%" y="100%">Viewer does not support full SVG 1.1<\/text>//g' *.svg-->
<!--![](bg.png)-->

行存更适合结构化数据，传统的关系型数据库基本都是行式存储，这样无论是在事务的支持还是在多表关联的场景下都能很好地发挥作用。而列存则比较适合非结构化或半结构化数据，只需要进行特定的查询，比如上表中行存有三个字段，而此时只需要查出 name 这列的数据，因此只返回一列的查询无疑效率是最高的，在数据量很大的情况下可以减少 IO，同时由于每列的数据类型都是一样的，我们还可以针对不同的数据类型进行压缩的优化，在查询时降低带宽的消耗。

HBase 在设计上有一个列簇的概念，那么当一个列簇下有多个列时，可以说此时 HBase 在逻辑存储上是行存的；若是一个列簇一个列，则可以说是列存的，但其在物理存储上都是 KV 结构，因此 HBase 其实是一种支持自动负载均衡的分布式 KV 数据库。

## 数据模型

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/DataModel-1737346423338.png)

对应的物理存储模型

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/PhysicalDataModel-1737346429742.png)


## 架构图

![](https://cdn.jsdelivr.net/gh/gleonSun/images@main/image/jg-1737346426371.png)

## 概念

- HMaster

- HRegionServer

- HLog

- BlockCache

- Region

- Store

- MemStore

- HFile