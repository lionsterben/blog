---
layout:     post
title:      "The Google File System"
subtitle:   "paper reading"
date:       2018-07-31 10:48:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - 技术随想
---

## 文件系统需求
- 将部件失败看作正常事件而不是异常
- 存储在GFS上的文件相比于传统操作系统大了很多，所以在文件分块大小需要重新考虑
- 大多数文件被修改时是通过append，而不是覆写已存在的数据
- 结合真实应用需求设计文件系统，应用需要文件系统支持

## 设计假设
- 构建在一般商用硬件上，需要一直监控，捕捉错误，容忍失败和恢复
- 系统存储一定数量的大文件。大文件大小在100MB以上，系统可以存储小文件，但是对小文件不进行优化
- 工作负载主要分为两块读取，大型流式读取和小型随机读取
- 工作负载还包括大量大型，顺序型对文件进行增加数据，一旦数据写入系统，就极少被修改
- 系统应该有效支持多个客户终端对同一文件并发增加数据
- 高容量稳定带宽比低时延重要许多

## 系统接口
GFS提供一个近似普通文件系统接口，但是并不是标准API如POSIX。文件在目录中按照层次结构组织，被路径名称唯一标记。系统支持create, delete, open, close, read,and write files。加上snapshot和之前提到的record append功能。

## 系统架构
<br>GFS集群包括一个master和多个chunkservers(和mapreduce很像)，每一个只是普通机器上运行的用户级进程。同时在一个机器上运行客户端程序和chunkserver非常容易实现。<br/>
<br>文件被切分为固定大小的chunk，每一个chunk都被唯一全局、不可变的64位chunk handle标记（被master在chunk生成时赋予）。chunkserver把这些chunk当作普通linux文件存储，通过chunk handle和字节范围读取。为了可用性，每一个chunk被在多个chunkserver上复制<br/>
<br>master上存储着系统元数据，包括命名空间，权限控制信息，文件到chunk的映射和chunks当前位置。还有一些系统间活动。master会周期性地利用心跳机制收集每个chunkserver地信息<br/>
<br>GFS client code 会代表应用利用文件系统API与master和chunserver进行通信。client和chunkserver都不会cache文件数据。<br/>

## 模拟流程
