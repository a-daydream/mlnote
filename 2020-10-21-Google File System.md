---
layout:     post
title:      Google File System
subtitle:   summary of The Google File System
date:       2020-10-21
author:     dreamStarRiver
header-img: img/emiliya.png
catalog: 	 true
tags:
    - distribute file system 
    - google file system
---

# [The Google File System](https://dl.acm.org/doi/pdf/10.1145/945445.945450)

> [典型分布式系统分析：GFS](https://www.cnblogs.com/xybaby/p/8967424.html#_label_2)

## Abstract

> 目标是为了大型分布式数据密集型的应用创建一个可扩展的、可在廉价商业硬件上运行容忍错误，给大量用户提供高的总性能的分布式文件系统。
>
> 吸取了之前分布式系统的构建目标，但是GFS项目由google应用的工作负担、技术环境所驱动，因此与之前的分布式系统有显著的区别，重新审视了传统的选择并且探索了不同的设计要点。
>
> 这篇论文将展示支持分布式应用的文件接口扩展设计，讨论了系统很多方面的设计。

## 1. Introduction

> 设计理念
>
> 1. 故障是常态而不是意外，因为系统构建在大量廉价商业硬件上并且服务与大量的客户。不间断检测、错误探测、错误容忍、自动恢复需要是系统必不可少的部分。
> 2. 文件相比于传统的标准大很多。因此需要重新设计I/0操作和块大小等假设和参数。
> 3. 大部分文件的操作是附加而不是重写。
> 4. 应用协作和文件系统API通过增加灵活性使整个系统收益。

## 2. Design overview

### 2.1 Assumptions

> 1. 系统是构建在普通的、廉价的机器上，因此故障是常态而不是意外
> 2. 系统希望存储的是大量的大型文件（单个文件size很大）
> 3. 系统支持两种类型读操作：大量的顺序读取以及小规模的随机读取（large streaming reads and small random reads.）
> 4. 系统的写操作主要是顺序的追加写，而不是覆盖写
> 5. 系统对于大量客户端并发的追加写有大量的优化，以保证写入的高效性与一致性，主要归功于原子操作record append
> 6. 系统更看重的是持续稳定的带宽而不是单次读写的延迟

### 2.2 Interface

> 提供与文件系统相似的接口，并没有实现标准的POSIX API，支持常用操作(create,delete,open,close,read,write)
>
> 设计了快照和record append(允许大量用户同时进行同一文件的附加操作)

### 2.3 Architecture

![gfs architecture](./gfs architecture.png)

> 1. 一个GFS集群由单个master和多个chunkservers组成，可以被多个客户访问。只要能够获得机器资源许可并且可以接受由不稳定代码导致的低稳定性就能够在机器上同时允许chunkserver和client。
> 2. 文件被拆分成固定大小的chunks，每个chunk由chunk handle标识(chunk handle在master创建chunk时确定)。chunkservers将chunk当作linux file保存，读写操作针对的chunk由chunk handle和byte range确定。可靠性由复制副本保证，默认保存三分在不同的chunkserver。
> 3. master保存所有文件系统元数据，包括名称空间，权限控制信息，文件到chunks的映射，chunks现在的位置。它同时控制系统范围的活动例如chunk租赁，孤儿chunks的回收，chunks在chunkservers之间的迁移。master通过心跳消息给出指令和收集chunkserver状态。
> 4. GFS client连接到每个应用实现文件系统的API并且负责和master、chunk通信来读取写入数据。clients和master交互进行元数据操作，但是所有承担数据的通信都直接和chunkserver进行。没有提供posix api因此不用进入linux vnode层
> 5. chunkservers和cilents不缓存文件数据，这简化了client并且整个系统排除了缓存整合的问题(cient会缓存元数据)。chunkservers不缓存文件数据因为chunks像本地数据一样存储，linux的缓存已经存储了经常访问的数据。

### 2.4 Single Master

> 单个master的结构简化了系统设计并且使得master能够利用全局数据去做复杂的chunk置换和复制决定，但是必须减少对它读写的调用使得它不会成为瓶颈。client从来不会从master读写文件数据，client只会从master知道需要访问的chunkserver，然后缓存这个信息一段时间之后直接和chunkserver进行交互进行接下来的操作。

**图表1的简单读操作步骤如下：**

1. client通过固定的chunk size，将文件名和byte偏移转换成文件的chunk index
2. client给master发送包含文件名和chunk index的信息，master将对应的chunk handle和复制块的位置返回给client。client缓存该信息用文件名和chunk index作为key。
3. client给最近的一个有该复制块的chunkserver发送请求，请求包含chunk handle和byte range。后续再读相同的chunk不需要和master交互了知道保存的信息过期或者文件再次打开。实际上client在一次对master的请求中是对多个chunks的请求，减少了与master后续的交互。

### 2.5 Chunk Size

>GFS chunk size 是64MB，普遍大于其他文件系统，每个chunk replica都以linux file的形式存储，并且按需扩展size

**较大chunk size的缺点**

1. 容易出现内部碎片，这个问题通过Lazy space allocation策略解决(后续追加扩展文件)。
2. 一些文件只有少许或者一个chunk，遇到多个clients的频繁访问会形成hot spots问题，目前的解决方案是将热点文件复制到更多机器上分担访问流量。

**较大chunk size的优点**

1. 减少clients和master交互的需要
2. client更可能在同一个chunk上执行更多操作，通过保持一个tcp链接减少网络负担
3. 减少了需要保存在master上的元数据

### 2.6 metadata





