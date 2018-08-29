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

# 失败处理
<br>对于每个map task和reduce task master都会维护一个状态：未运行，运行中，已完成<br/>
<br>master会定期Ping各个worker，如果worker失败的话，在它上面已完成的map task重设为未运行，正在运行的map task和reduce task都设置为未运行，全部重新运行。已完成的map task之所以要重运行是因为中间结果保存在worker里。<br/>
<br>master失效，从保存点恢复（那为保存点之后的操作怎么办？也保存了吗？），或者直接重新执行job<br/>
<br>当执行完map task的机器失效后，master在另一台机器上运行，reexecution会通知所有在执行reduce task的worker，如果worker还没有从A上读取data，就直接从B上读取<br/>
<br>关于任务重复导致的结果文件一致性问题，map task重复，后完成的发消息给master，master直接忽略。reduce task重复，利用文件系统重命名操作的原子性，保证只有一份。<br/>
# M和R选择
master需要做O（M+R）规划决定，O（M*R）存储状态空间需求（但是要求容量很小），一般选择M使得单独任务处理数据规模在16MB到64MB中间，R是worker machine数量的几倍左右

# 改进
- 在一个mapreduce操作快结束时，观察还有哪些任务一直没结束，重新运行这些任务
- 可以自定义hash function
- 中间值pair按key排序（这不是原始就有的吗）
- combiner function：在map端将可以合并的（key，value）pair合并，直接使用reduce function，除了输出变一下
- 支持自定义输入输出格式
- 副作用：会生成很多附加文件（原子性生成），第二段没看懂。。。
- 支持跳过不能读取的记录 
- master运行一个http server，支持直接从网页端看到运行情况、生成文件和错误信息
- counter计数器，查看运行情况

## 实验想法
- map  phase和reduce phase都尽量调用所有机器，不存在在开始分为map machine和reduce machine，map task执行完毕结果刷到磁盘里，同时把位置通知master，master启动reduce machine，通过这些位置读取数据。