---
layout:     post
title:      "GC标记-清除算法"
subtitle:   " \"Mark Sweep GC\""
date:       2019-06-26 19:20:00
author:     "Clinat"
header-img: "img/post-bg-windchimes.jpg"
catalog: true
tags:
    - GC
---

> “本篇为《垃圾回收的算法与实现》一书的学习笔记。”


## GC标记-清除算法

标记清除算法，顾名思义，该算法包括两个部分，标记阶段和清除阶段。

**标记阶段：**标记阶段，该算法会将root指向的对象进行标记，之后递归调用，同样将这些对象指向的其他对象进行标记，标记之后的状态如下图所示。标记的位置在对象头中。所以该算法的耗时与活跃对象的数量成绩正比。

<img src="/img_post/MarkSweepGC/marksweep0.png" style="zoom:50%">

**清除阶段：**在标记阶段结束之后，该算法会遍历堆中的所有对象，根据标记阶段的标记判断这些对象是否为活跃对象，将非活跃对象清除，并添加到空闲链表中，所以该阶段的耗时和堆的大小是成正比的。

<img src="/img_post/MarkSweepGC/marksweep1.png" style="zoom:50%">

除了标记阶段和清除阶段，还有内存分配和合并的过程，分配的过程便是遍历空闲链表，找到满足条件的内存块，其中分配方式分为三种，分配第一个满足条件的空闲块，分配最小的满足条件的空闲块和分配最大的满足条件的空闲块。合并是指将相连的空闲块合并成为一个更大的空闲块。



## 优缺点

**优点：**该算法的最大的特定便是实现简单，同时由于该算法不涉及移动对象的存储位置，所以非常适合保守式GC算法。

**缺点：**由于算法实现非常简单，便必然会有很多缺点。该算法会导致内存碎片化；同时，每次分配内存时，都要遍历空闲链表，效率较低；同时该算法不支持写时复制技术。



## 多个空闲链表

为了提高分配空间的效率，采用多个空闲链表的办法，将特定大小的空闲块放到指定大小的空闲链表中，这样每次分配空间只需要遍历特定的链表而非整个空闲块组成的链表。

<img src="/img_post/MarkSweepGC/marksweep2.png" style="zoom:50%">



## BiBOP方法

BiBOP是Big Bag Of Pages的缩写，该方法的目的是消除碎片化，该方法堆内存的分配方式是将堆划分成多个固定的块，每个块只分配固定大小的对象。但是如果分配的对象大小不均匀导致某个块种只分配了极少数的对象，就会导致堆的利用率降低。同时，在多个块中分散着相同大小的对象也会导致堆的利用率降低，如下图。

<img src="/img_post/MarkSweepGC/marksweep3.png" style="zoom:50%">



## 位图标记

GC的标记清除算法对对象进行标记时，各个标记的位在对象头中，所以需要对将对象和头一同进行处理，所以与写时复制技术不兼容，所以采用位图标记的方式，将标记记录在位图中，这里应该注意的是标记在位图中的位置和对象头在堆中的位置是相对应的，如下图。

<img src="/img_post/MarkSweepGC/marksweep4.png" style="zoom:50%">

之前对对象进行复制的开销是非常大的，使用位图之后，对位图的修改会导致位图的复制，但是位图的大小非常小，所以是可以接收的。

**写时复制技术：**在Linux系统中对进程进行复制操作，即执行fork()方法，大部分的内存空间都不会被复制，而只是将子进程共享了父进程的内存空间。即将子进程的虚拟地址空间映射到父进程的物理地址空间。

而此时所说的标记清除算法与写时复制技术不兼容是指，如果执行fork()方法对进程进行了拷贝操作，则在进行GC的过程中，该算法会修改所有对象的头，因为标记位在对象头中，所以写时复制技术会认为对这些对象进行了修改，则会对这些对象进行复制，这便提现不出写时复制技术的优势，而采用位图的方式，由于位图大小非常小，所以对其进行复制操作的开销是非常小的。



## 延迟清除法

前面说过，清除阶段的耗时是和堆的大小成正比，为了减少这一阶段的耗时，采用延迟清除法。

该方法是指，在标记阶段完成之后，并不立即执行清除阶段，而是等待下次内存分配，执行清除操作，如果能够通过清除操作获取足够嗯内存，则直接分配内存。如果不能分配内存，再执行标记操作，之后再执行清除操作并分配内存。

但是如果堆内存的分配情况如下：

<img src="/img_post/MarkSweepGC/marksweep5.png" style="zoom:50%">

活跃对象和非活跃对象均分配到了一起，就会导致"垃圾堆"，这时当延迟清除扫描活跃对象所在的区域时就会增大应用程序的暂停时间，因为延迟清除扫面的起始位置为上一次扫描的结束位置，所以极端情况下，会出现这种情况。。