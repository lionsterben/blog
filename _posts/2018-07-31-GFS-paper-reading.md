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
<br>简单描述一下客户端请求GFS的数据流程。1.client知道固定的chunk size，根据这个数据计算所需要数据所在文件的chunk index。2.将文件名字和chunk index作为请求发送给master，master回复给client chunk handle（全局64位唯一标识）和chunk副本所在的位置,client会用file name和chunk index作为key将这个信息cache。3.客户端选择离它最近的副本位置，对这个chunk server发送请求，将chunk handle和byte range发送，等待收到数据就行了。<br/>

## 元数据
<br>master主要存储三类元数据：file和chunk的命名空间、file到chunk的映射、每个chunk副本的位置。前两种存储在内存中，这两种数据通过operation log与其它副本机器保持一致。chunk副本的位置不会和真实数据保持一致，它利用心跳机制询问chunk server的信息。<br/>

<br>operation log记录metadata变化事件。构造并发事件的逻辑时间顺序（分布式系统的全序？lamport老爷子的论文看不懂啊，摔）。file和chunk被创造它们的逻辑时间唯一标识（chunk handle）。operation log非常重要，如果不进行备份，一旦丢失就全没了。当备份operation log到多个机器上时，只有当operation log刷到所有机器上磁盘上时，才通知client操作成功。operation log需要保持很小以便快速重启master，所以会当log超出一定大小时，会checkpoint operation log。<br/>

## 一致性模型
<br>说实话，这一章我看得模模糊糊的，只将我的理解大概记录下来。一致性模型表示多个节点副本数据一致性（不确定，再学习学习），GFS采用的是relaxed consistency，不保证强一致性，client可能看见不同数据在不同的副本上。（？）对file namespace的改变是原子性（锁机制和operation log逻辑顺序）。关于文件的改变分为append和write，并发append是define的，并发write是consistent。append采用的是至少原子性添加一次语义而且是在GFS选择的偏移处，所以会出现重复和padding。（具体见论文2.7节，内容太多，还需要学习）。因为client会cache chunk位置信息，所以有可能读取过期数据。<br/>

## 数据改变流程
<br>GFS用Lease（租约）方法，也就是给众多副本中其中一个证书，证明它是primary，这个primary副本选定mutations的顺序，并推送给所有副本。1.client询问master哪一个chunkserver保存当前chunk的lease和其他副本的位置，如果没有lease的话，master挑一个给。2.master回复这些信息给client。client cache这些信息，只有当primary不可达或者没有lease时候，才需要重新获得。3.client push data给所有副本，发送这些数据可以以任意顺序。4.当所有副本接收到数据后，client会向primary发送写请求，primary对这些数据（很可能来自多个client）进行连续标号，之后按照序号进行apply mutation。5.primary发送写请求给所有其它副本，它们按照primary同样的顺序进行apply mutation。6.副本完成后回复primary表示它们已经完成。7.primary通知client完成。如果失败的话，很大可能是primary成功，这样的话重复3到7步。<br/>
<br>GFS将控制流和数据流解耦了，控制流由client流向primary再流到all secondaries。数据流为了最大化利用带宽，使用chain式传输，每台机器会将数据传到当前网络拓扑离它最近的机器（还没有收到数据）。利用TCP传输，当收到数据时立即就可以发送出去了。<br/>

## Atomic Record Appends 
<br>传统并发写文件无法被serializable（好像不是rpc里的序列化），简单来说，无法控制顺序，导致被写的区域被零散的数据片段填充，不同的顺序会出现不同的片段，而且不知道是哪个client的数据。GFS提出record append，在指定的offset至少写入数据一次。record append的修改数据方式遵从上一节的逻辑，只需要在primary增加点逻辑，如果primary的chunk size不足，pad到最大容量，通知所有secondaries。同时对next chunk重复这些操作。如果record append在某个副本失败，client会重复这个操作，（As a result, replicas of the same chunk may con-tain different data possibly including duplicates of the same record in whole or in part. GFS does not guarantee that all replicas are bytewise identical. It only guarantees that the data is written at least once as an atomic unit.3.3section）（In addition, GFS may insert padding or record duplicates in between 2.7section）论文原文这个，表明重复操作会导致多个重复数据，可是之后又说明所有的副本会在相同的offset至少写一次，这样的话即使重复也会被覆盖啊，如果是有重复数据，也是2.7节提出的在两个offset数据之间填充重复数据，同时论文指出这样操作能够保证所有副本都会有相同的end of record。（一个坑，待填）

## Snapshot
<br>snapshot操作能够几乎在瞬间拷贝一份文件或者目录树，GFS利用了copy on write技术。在master收到对文件snapshot时，首先对文件的chunk全部撤销lease，这样之后master在之后对文件写的话，能够首先创造一份chunk的copy。当lease被撤销之后,master将snapshot的log刷进disk，然后执行操作，这个新建立的文件指向同源文件相同的chunk C。当client想写chunk C，master会寻找当前lease所在的副本位置，master会发现指向chunk C的文件超过一个，master会首先新建一个新的chunk handle C'，然后要求每个有chunk C 的chunk server copy一个同样的chunk C'副本，然后在chunk C'赋予lease，修改，返回给client。<br/>

## MASTER OPERATION
<br>master会执行所有namespace的操作，还有管理系统的chunk副本（决定chunk副本放置位置，保证chunk副本数量足够，重平衡chunk server负载，回收垃圾空间）。<br/>
<br>许多master operation会需要很长时间：snapshot就需要撤销所有需要snapshot的chunk的lease，所以对这些namespace运用锁机制。GFS的namespace结构就是树形结构，map full pathnames to metadata。论文对读写锁的运用有详细的阐述，我就不举例子了。<br/>
<br>副本放置问题实际上是data reliability、availability和网络带宽利用之间的tradeoff。<br/>
<br>创建chunk副本有三个原因，创建chunk、重复制chunk、重平衡。创建一个chunk选取chunk副本时候，有三个考虑因素，1.将副本放在低于平均磁盘空间利用率的chunk server里。2.限制对同一chunk server在某个时间段的create次数。3.上一段要求。<br/>
<br>重复制发生在副本数量小于规定数量（有各种各样的原因导致这个状况发生），重复制也有优先级，第一个是距离规定数量的大小，第二个优先活跃的文件chunk，第三个优先阻塞client进程。复制时候按照create的条件选择副本位置，同时为了防止流量将client跑崩，限制cluster整体和每个chunkserver的流量。<br/>
<br>master会周期性地重平衡副本位置，标准和之前一样。总之，目标就是平衡所有chunk server的磁盘利用率。<br/>

## Garbage Collection
<br>当文件被删除后，不会立即回收它们的物理存储空间，会等到在文件和chunk层级进行垃圾回收在进行。<br/>
<br>当一个文件被删除后，master会立即log，file会被一个隐藏名字（利用删除时间戳标记）重命名，在master日常扫描文件系统namespace，会删除那些超时3天（可设置）的文件。这样操作可以在一定时间段内重新读取删除文件。对待chunk namespace同理，master会定期扫描过期chunk（无法被任一file访问到的chunk），删除它们的元数据，通过chunk server和master的心跳机制，要求chunkserver删除这些副本。<br/>
<br>GFS利用chunk version number进行过期副本检测，每次对chunk进行lease赋予后，就增加chunk version number，master和最新的副本都会保存这个数字。master会通过chunkserver心跳机制发送的number判断是否过期，如果发送的数字比自身的还大，就说明master在赋予lease时候，chunkserver失效了，将这个副本作为最新的同步。master会在垃圾回收时候清除这些过期副本，在这之前，回复client chunk位置时当作它们不存在。这样做的一个好处时，master在回复client chunk副本位置时，会附带给version number，这样client在读取时候，就会校验一下，保证读到最新数据。<br/>

## FAULT TOLERANCE AND DIAGNOSIS
<br>失败容忍性是分布式系统关键特性。其中几点在之前的系统概述都已经提及，简单总结一下。1.快速恢复，master和chunkserver。2.chunk副本。3.master副本。<br/>
<br>数据完整性检测。数据完整性和TCP之类的蛮像，将每个chunk里的数据分为64KB的小块，每个小块都有一个32位的checksum，和元数据一样，这个数据会被master保存在内存里，并且用log永久保持。每次读取时候，chunk server都会计算需要读取的chunk的chunksum对照。checksum对于record append重度优化，每次加进去数据，只需要对加进去的block计算，write的话需要对所有被修改的数据重新计算。<br/>
<br>分析工具。GFS大量记录machine的事件，打成log。GFS server记录chunkserver启动和下线，所有RPC请求和回复。这些log可以随时被删除，但是我们在不影响空间要求时，尽量保存它们。<br/>


