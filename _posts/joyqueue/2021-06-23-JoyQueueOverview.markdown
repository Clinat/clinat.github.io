---
layout:     post
title:      "JoyQueue概述"
date:       2021-06-23 18:26:00
categories: 技术
tags: JoyQueue
---

开篇诗两句：

​	**泽国江山入战图，生民何计乐樵苏。**

​	**凭君莫话封侯事，一将功成万骨枯。**	——曹松《己亥岁》

<br/>


## GitHub

（1）JoyQueue：https://github.com/chubaostream/joyqueue 

（2）JournalKeeper：https://github.com/chubaostream/journalkeeper 


<p id = "build"></p>
## Kafka和JoyQueue对比

JoyQueue支持了存储分离，这里只是对支持存储分离之前的JoyQueue进行对比

### （1）存储模型

JoyQueue在存储模型上的设计结合了Kafka和RocketMQ的存储设计，针对每个partition，只有一个日志文件，但是可以针对日志文件建立多个索引文件，以此来实现多个分区，同时支持多个消费者同时消费。 

Kafka的每个partition只有一个日志文件，并且只有一个索引文件，每个partition只能支持一个消费者消费。 



JoyQueue为了提高消息的写入和消费速度，自己实现了内存池，消息写入时，首先从内存池中申请内存，然后写入到内存中，然后异步刷到磁盘上。并且提供了读写页和只读页，并且自己实现了清理逻辑。 读写页本质上就是directBuffer，只读页是mMapBuffer 

Kafka通过fileChannel将消息写入到pageCache上，然后由操作系统控制刷盘。 

### （2）数据一致性和副本同步

JoyQueue通过Raft协议实现副本之间数据同步和数据一致性的保证 

Kafka是通过副本节点的拉取线程，主动从主副本拉取数据来进行副本同步，数据一致性是通过高水位（HW），LEO以及leader epoch共同维护的 

### （3）吞吐量

JoyQueue单分区，没有副本同步的情况下，吞吐量高于Kafka。 

JoyQueue单个消费者的消费速度远低于Kafka单个消费者的消费速度，但是JoyQueue的单个分区可以多个消费者同时消费，Kafka单个分区只能有一个消费者进行消费 

JoyQueue的消费者不是数量越多消费速度越快。 

### （4）性能消耗

JoyQueue和Kafka在消息发送方面都会增加CPU负载，并且和消息流量相关 

JoyQueue消费者同样会增加CPU负载，和消息流量相关，但是Kafka的消费者基本不会增加CPU负载。 

​															


