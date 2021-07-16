---
layout:     post
title:      "Linux常用命令"
date:       2021-06-29 11:08:00
categories: 技术
tags: Linux
---

开篇诗两句：

​	**李杜诗篇万口传，至今已觉不新鲜。**

​	**江山代有才人出，各领风骚数百年。**	——赵翼《论诗》

<br/>

## 查看线程占用CPU情况

（1）：查看占用CPU最高的线程ID

```shell
top -Hp pid
```

H打印线程信息，p指定pid（pid可以通过jps命令查看）

<img src="/assets/linux/command01.png" style="zoom:50%">

（2）将线程ID转化为16进制，使用 jstack -l <pid> | grep 0x1bd9 -A 10 查看对应的线程

```shell
jstack -l <pid> | grep 0x1bd9 -A 10
```

<img src="/assets/linux/command02.png" style="zoom:50%">

## 后台运行程序，并指定日志目录

```shell
nohup sh ./kafka_2.13-2.8.0/bin/kafka-server-start.sh server.properties > /opt/meituan/apps/mafka/kafka.log &
```

