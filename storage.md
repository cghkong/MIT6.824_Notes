# 分布式存储

## GFS
GFS是为大规模分布式数据密集型应用程序而设计的可伸缩性（scalable）的分布式文件系统。GFS为在廉价商用设备上运行提供了容错能力，并可以在有大量客户端的情况下提供较高的整体性能。

## 假设
1. 系统由许多可能经常发生故障的廉价的商用设备组成，因此它必须具有持续监控自身并检测故障、容错、及时从设备故障中恢复的能力

2. 系统存储一定数量的大文件(100MB或几GB级别)，这些文件需要被高效管理。系统同样必须支持小文件

3. 系统负载主要来自两种读操作：大规模的流式读取和小规模的随机读取。性能敏感的应用程序通常会将排序并批量进行小规模的随机读取，这样可以顺序遍历文件

4. 系统负载还来自很多对文件的大规模追加写入，一旦写入几乎不会更改。系统同时也支持小规模随机写入，但并不需要高效执行

5. 系统必须良好地定义并实现多个客户端并发向同一个文件追加数据的语义。最小化原子性需要的同步开销非常重要

6. 持续的高吞吐比低延迟更重要

## 架构
一个GFS集群包括单个master（主服务器）和多个chunkserver（块服务器），并被多个client（客户端）访问。

![image](https://user-images.githubusercontent.com/79254572/182100338-f397aeac-0960-4f68-8d7f-2a9d3f946513.png)

文件被划分为若干个固定大小的chunk（块）。每个chunk被一个不可变的全局唯一的64位chunk handle（块标识符）唯一标识，chunk handle在chunk被创建时由主节点分配。每个chunk存储在本地的磁盘，通过chunk handle和byte range来访问数据。通常为了可靠性，需要对数据进行拷贝副本，可以为副本指定级别(primary、secondary)。

## master
master中维护系统所需的元数据信息：命名空间（namespace）、访问控制（access control）信息、文件到chunk的映射和chunk当前的位置。此外，master还会控制系统活动如chunk的租约、垃圾回收以及迁移等。master周期性地通过心跳（HeartBeat）消息与每个chunkserver通信，向其下达指令并采集其状态信息。

GFS client的代码实现了文件系统API并与master和chunkserver通信，代表应用程序来读写数据。client与master交互，但是所有的数据直接与chunkserver交互。

chunkserver和client都不需要缓存数据，大部分应用程序需要流式地处理大文件或者数据集，缓存没有用武之地，同时还能消除缓存一致性问题，简化系统设计

### chunk设计
chunk大小选择64MB，采用懒式空间分配避免内部碎片

#### 优点：
   1. 减少了client与master交互的次数
   
   2. client更有可能在一个chunk上执行更多的操作，通过与chunkserver保持更长时间的TCP连接来减少网络开销
   
   3. 减少了master中保存的元数据大小，这样可以将元数据保存在master的内存中。

#### 缺点：
   1. 如果多个client访问同一个文件，那么存储这这些文件的chunkserver会成为hot spot（热点）

   2.对管理小文件不利
   

### 元数据
元数据主要有三种：

1. 文件和chunk的命名空间（namespace）

2. 文件到chunk的映射

3. chunk的每个副本的位置

前两种需要记录到日记操作中持久化到磁盘并远程备份，master不会持久化存储chunk的位置信息，而是在启动时和当chunkserver加入集群时向chunkserver询问其存储的chunk信息。

master将元数据保存在内存中，可以使master周期性扫描整个集群的状态变得简单高效，对于实现垃圾回收、chunkserver故障时重做副本、chunkserver间为了负载均衡和磁盘空间平衡的chunk迁移提高便利。

master控制着所有chunk的分配并通过周期性的心跳消息监控chunkserver状态，启动时从chunkserver获取chunk的位置信息。

master通过持久化和备份操作日记来进行故障恢复，通过重放checkpoint之后的日记来恢复系统状态

### 一致性模型


## 系统交互


## master控制操作



## 容错

