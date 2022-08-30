# kafka-notes
> kafka的学习笔记  https://github.com/pierre94/kafka-notes

16年初识Kafka时，我接触的还是0.8.x版本。不知不觉Kafka已经发展到目前的2.x.x版本。
笔者也见证了 Kafka的蜕变，比如
- 旧版客户端的淘汰
- 新版客户端的设计
- Kafka 控制器的迭代优化
- 私有协议的变更
- 事务功能的引入
……

Kafka从昔日的新星逐渐走向成熟，再到王者地位不可撼动，这期间有太多的故事可讲，有太多知识可以学习。

## 源码阅读
在阅读《深入理解kafka:核心设计与实现》这本书的时候，愈发对kafka的源码和内部实现感到兴趣。按照书中大纲和kafka社区相关资料，启动对现网主流版本2.2.0的源码进行深入理解。

- 源码笔记: [kafka-2.2.0](./kafka-2.2.0-src)

### 一、核心设计思想
> https://kafka.apache.org/documentation/#design

#### 0、 Motivation-设计初衷
- 高吞吐
- 低时延

#### 1、Persistence-持久化

#### 2、Efficiency-效率/性能

#### 3、producer相关设计与实现

- [X] producer流程全流程概述

#### 4、consumer相关设计与实现


- [ ] 消费组分区分配策略
- [ ] [`ConsumerCoordinator`与`GroupCoordinator`](./ConsumerCoordinator与GroupCoordinator.md)
- [ ] [`__consumer_offsets`剖析](./__consumer_offsets剖析.md)

#### 5、消息传递语义 

#### 6、副本机制

#### 7、日志Compaction

#### 8、Quotas-配额

### 二、关键实现

#### 1、网络层

#### 2、消息层

#### 3、消息格式
- [ ] 日志格式的演变

#### 4、日志
- [ ] 日志存储逻辑

#### 5、分布式

#### 其他
- [ ] 间轮设计与实现分析

- [ ] 延时操作设计与实现分析

- [ ] 控制器设计与实现分析 | 难点

- [ ] 副本相关逻辑
    - [ ] 副本基本原理
    - [ ] 副本同步机制与可靠性分析

### 其他
- [X] [Zero Copy相关](./ZeroCopy.md)
- [X] [kafka刷盘机制](./kafka刷盘机制.md)


> 事务目前没有大规模使用到，优先级低。Kafka 的事务可以看作Kafka 中最难的知识点之一!目前先写一下简单介绍,后续再补上源码级分析。
- [ ] [事务实现](./事务.md)

### 四、周边功能
- [ ] console producer 与 consumer
- [ ] kafka streams
- [ ] kafka mirror
- [ ] kafka connector

### 五、问题分析与解决
- [X] [` __consumer_offsets`部分分区异常导致消费不到数据问题排查](./__consumer_offsets部分分区异常导致消费不到数据问题排查.md)
- [X] [使用kafka-connector消费数据时看不到consumer-id等信息](./使用kafka-connector消费数据时看不到consumer-id等信息.md)

### 六、功能进阶
- 过期消息
- 延时队列
- 消息路由

## MISC
#### kafka平滑升级方案
- [携程 Kafka 升级到2.0的实战经验](https://cloud.tencent.com/developer/news/377416)


## 参考资料
- 《深入理解kafka:核心设计与实现》

## 微信公众号
更多文章与内容请关注:
![](https://bear2-10045049.cos.ap-shanghai.myqcloud.com/mzmq5nkwlf.png)
