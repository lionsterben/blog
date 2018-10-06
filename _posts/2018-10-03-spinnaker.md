---
layout:     post
title:      "Using Paxos to Build a Scalable, Consistent, and Highly Available Datastore"
subtitle:   "paper reading"
date:       2018-10-03 19:22:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---
## Abstract
<br>Spinnaker是IBM和linkedin的学者提出的一个大规模分布式数据仓库，提供key-value存储格式和get-put api，在读取的时候提供强一致性和时间轴一致性。整个系统建立在PAXO协议上（可以替换为Raft协议），因此具有极强的失败容忍性。<br/>
<br>传统的数据库副本模式是master、slave模式，很像之前博客介绍vmware VM冗余模式，和vmare有相同的缺陷就是会丢失数据，vmware那篇只保证output是一致的，但是数据仓库系统需要保证apply的指令在之后的系统里都存在，论文举了一个例子，展示master、slave模式达不到这一点，这时候就需要paxo或者raft协议。分布式系统里的CAP理论，CAP分别指Consistency、Availabilty、Partiton tolerance([wikipedia](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86))分布式系统最多只能满足两个要求。Spinnaker假设在单个data center里，所以放弃Partiton tolerance，选择CA。关于一致性，Spinnaker提供强一致性和时间轴一致性，强一致性是指在提供服务时，任意副本的数据是相同的，时间轴的一致性可能会读到过期数据，但是所有的副本会按照相同的顺序执行，达到最终一致性，具体实现放在之后再说。<br/>

## Architecture
<br>Spinnaker将文件存储为Key、value格式，按照key  range分布在不同Node存储，假设需要N份副本，那么确定key range的开始节点，那么就在之后的N-1个节点进行复制，论文默认N=3。所以node内部的框架就很重要了，和raft实现很像，首先是log数组存储接受的 write log，commit queue存储已经commit的write log准备apply，其他就是存储key、value的位置和一些为了recovery和leader election的空间了。注意的是每个node不只是一个key range数据，因为顺序复制的原因，一个node里应该是N个key range副本，所以论文为了性能需要，把log全部放在一起了，但是用不同的逻辑LSN分开。zookeeper提供中心化的位置存储所有node的元数据，用于leader election等事件，注意zookeeper不参与read和write，它只和node之间通过heartbeat交换信息获取node的元数据。<br/>
<br>关于复制协议，相邻N个node存储相同的Key Range组成一个cohort，论文提出的类paxo协议就是在每个cohort运行的，讲真，和raft简直一样，我就不再说一遍了，有兴趣的同学可以看[这篇](https://lionsterben.github.io/lionsterben/2018/09/03/Raft_part1-paper_reading/)。这里说一下strong consistency和timeline consistency，强一致性就是读的时候只和leader交互，保证读到的是最新的，而时间轴一致性可以读任一node，所以可能读到还没有write的数据。<br/>

## Failure Recovery
<br>这和raft里不太一样，或者说目的一样只是讲法不同，在raft里通过append entries自然地使得follower跟上leader的节奏，request vote使得leader失效后，自己选择合适的leader，发送heartbeat，使得follower跟上新leader。而论文则是认为将follower失败后跟上leader分为两个阶段：1.local recovery，在自己的checkpoint reapply log直到committed LSN 2.catch up，在committed LSN到last LSN之间的log则是不确定的，不知道leader是否commit，所以需要leader发送从自己的committed LSN到leader最后的log，但是leader这时候需要锁住写要求，让follower能够catch up，这就不如raft的实现了，raft通过周期性的心跳，在不影响leader的情况下使得follower能够catch leader。论文里面还说了一个坑，之前提到的node log是在一起的，所以在follower覆写或者抛弃commited LSN的log时，需要维护一个skip list，跟踪每个cohort log的位置。<br/>
<br>如果是leader失败，产生新leader之后，leader需要为所有follower发送f.cmt(committed LSN)到leader.cmt的log，再发送commit信息，让follower先到leader.cmt这块，等到至少有一个回复成功(N=3)，之后再发送leader.cmt到leader.lst(last LSN)，按照正常commit操作，最后open cohort开放写请求。<br/>

## Leader Election
<br>Spinnaker使用zookeeper进行选举，zookeeper的论文我还没看，所以这里就大致讲下这篇论文实现。每个node都运行一个zookeeper client来与zookeeper交互，假设r对应zookeeper里某个cohort进行leader election的节点，某个node开始选举的时候，先通过client将r的旧状态清掉，然后把自己的标记n和last LSN填入一个临时节点放在/r/candidate下面，并且watch这个节点（如果有变化，就通知client），等到majority cohort node出现在/r/candidate，每个cohort node都检查下自己是不是leader，leader是具有最大last LSN的node，如果n是leader而且/r/leader是空的，然后就将leader的hostname在/r/leader写入暂时节点，防止之后leader失效后，cohort可以被通知。然后就是上一节提到的leader发送follower消息了。失败的cohort node就直接读/r/leader获取leader信息。有些细节还不是很懂，等我看完zookeeper之后再说吧。<br/>