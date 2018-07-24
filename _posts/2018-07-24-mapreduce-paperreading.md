---
layout:     post
title:      "MapReduce: Simplified Data Processing on Large Clusters "
subtitle:   "paper reading"
date:       2018-07-24 16:50:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - 技术随想
---

# mapreduce流程
- 首先将待处理文件分为M部分，然后复制多份用户代码到集群机器上
- 选取集群机器其中一台为Master，其余为worker，有M个map任务和R个任务，master选择空闲的worker给它map或者reduce任务
- 执行map任务的worker读取文件一部分-->系统自动Parse为多个（key1，value1）pair，每个pair都输入到用户制定的map function里，生成中间（key2，value2）数据存在内存中
- map worker 周期性地将内存中的数据写入到disk中（存储方式利用hash方式，分为R个部分，存在R个分区），这些数据的地址会被存到master，master负责将这些地址送到reduce worker
- 当reduce worker被通知被分配给它分区（hash里的）的数据全部生成后，reduce worker从master那里拿到所有map worker属于该分区数据的地址开始读取，全部读取完成后，按照key排序
- reduce worker遍历已经排序好的数据，每当它遇到新的key值，就将属于同一个key的value提取出来，输入到reduce function里，结果写入output
- 当所有任务都完成后，master唤醒用户程序

## tips
- map worker接收的是文件的M部分之一，所以共有M个map task
- reduce worker接受的是所有map task生成的R部分之一，而R部分是map task对生成key的hash的结果。共有R个reduce task