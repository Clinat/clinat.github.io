---
layout: post
title: "JoyQueue源码-直接内存池"
date: 2021-06-23 19:44:00
categories: 技术
tags: JoyQueue
---

开篇诗两句：

​	**昨夜星辰昨夜风，画楼西畔桂堂东。**

​	**身无彩凤双飞翼，心有灵犀一点通。**	——李商隐《无题》

<br/>

## 直接内存池的结构

### 1、参数

<img src="/assets/joyqueue/directBufferParameter.png" style="zoom:50%">

### 2、PreLoadThread线程

<img src="/assets/joyqueue/directBufferPreLoadThread.png" style="zoom:50%">

### 3、EvictThread线程

<img src="/assets/joyqueue/directBufferEvictThread.png" style="zoom:50%">

## 参数方面

（1）UsedSize：表示内存池已经使用的内存，包括已经分配出去的内存（directBuffer和mMapBuffer）和内存池中已经存在但是还没有分配出去的部分 

（2）CoreMemorySize：内存池的核心内存数，当内存池的内存数量低于该值时，会尝试申请新的内存添加到内存池中。当UsedSize超过该值时，会尝试回收内存，放到内存池中。（主要是保证每个size的buffer的核心数量） 

（3）EvictMemorySize：清理阈值，当使用量超过该阈值时，就会开始清理内存，直到使用量低于该阈值 

（4）MaxMemorySize：内存池最大内存数量，当内存使用量超过该阈值时，不能继续申请内存，并尝试回收内存，如果依然不能减少使用量，则抛出OOM异常 



PreLoadCache，coreCount默认是3个，保证至少有3个预加载得内存块，maxCount默认为10，保证最多只能存在10个预加载的内存块，多余得会在清理线程中回收 

日志文件的文件大小为128M，索引文件的大小为512K，所以对应的内存块大小也是128M和512K 

## 数据维护

（1）directBufferHolders：直接缓冲区列表，表示内存池分配出去的直接缓冲区。

（2）mMapBufferHolders：mmap缓冲区列表，表示内存池分配出去的mmap缓冲区

（3）bufferCache：内存池维护的各个size的内存块，PreLoadCache类型，包括内存块的大小，核心数量，最大数量以及内存列表

## 线程方面

（1）PreLoadThread线程：主要作用是预加载各个size的内存块，当某个size的内存块数量少于核心数量，则会尝试创建该size的内存块。如果内存池的内存使用量低于核心阈值，则采用创建新的内存块的方式增加该size的内存块。如果内存池使用量高于核心阈值，则采用从已经分配出去的内存块中进行回收的方式增加内存块的数量。

（2）EvictThread线程：内存块清理线程，当使用量超过清理阈值才会执行。会从所有已经分配出去的内存块中，根据回收权重weights来排序，依次回收内存块，直到使用量低于清理阈值。

## 回收策略

回收策略采用的是LRU（最近最少使用）： 

权重： 

（1）只读页（mMapBuffer）：最近的访问时间 

（2）读写页（directBuffer）：最近的访问时间 + 额外权重 

额外权重默认是60s 

例如：一个只读页，上次访问时间是T；一个读写页，上次访问时间是T - 60s，那么这个两个页的权重是相同的 

回收的时候优先回收权重较小的页，保证**优先回收只读页** 

## 其他

（1）尝试回收buffer的逻辑在消息发送和消费中介绍

（2）创建对应Size的PreLoadCache的逻辑在其他笔记中介绍