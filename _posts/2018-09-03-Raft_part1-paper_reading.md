---
layout:     post
title:      "In Search of an Understandable Consensus Algorithm (Extended Version) Part_1"
subtitle:   "paper reading"
date:       2018-09-03 15:57:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - 技术随想
---

Raft是分布式系统中的共识算法，属于分布式系统里的基础，通过共识算法才能保证服务器失败容忍，即只要majority服务器还存活，系统就可以一直运行下去。Raft属于Paxo针对理解性的改良版，曾经我花了一上午时间看Paxo协议，结果只记得将军、间谍之类的东西了（误，这篇Raft写得更为详尽和清晰。由于Raft涉及太多细节性内容，在一篇博文很难解释清楚，所以本篇文章只将论文框架写出，具体的内容感兴趣的小伙伴可以直接读这篇论文为好。论文主要将raft分为leader election、log replication和safety三部分阐述，我也就分这几部分开始说。

## raft basics
<br>raft将在这个系统内的服务器分为三个角色：leader、candidate and follower。一个系统只有一个leader，正常情况其它服务器应该都是follower，在follower超时，需要竞选leader时候变成candidate。follower只能被动地接受leader的信息，leader负责处理client全部的请求。raft将系统运行时间分为任意长度的term，每个term以一次选举为开端（term被视作raft里的逻辑时钟，检测过时信息）。关于term检测和服务器三种状态的变换具体见论文的Figure 4，我就不赘述了。实际上所有的细节全部在Figure 2这张图里，理解raft务必仔细研究这张图。

## leader election
<br>leader通过周期性地心跳机制防止follower变成candidate，从而保住自己leader的位置。但是如果follower超时（也就是在某个时间段既没有收到leader的心跳也没授予其它candidate vote），变成candidate，term加一，准备选举，对所有其它request vote请求，这种情况只有三个结果：1.收到majority vote，变成leader 2.其它服务器变成leader，自己降格为follower 3.计时器过了，没有服务器变成leader。现在分别阐述每种情况发送的条件和后果，1 在相同的term收到majority的vote然后变成leader，之后直接发送heartbeat宣示主权。2 本candidate在等待vote时候，来了其它leader的heartbeat，自己就变为follower 3 多个follower同时变成candidate，开始选举，vote被分为多块，导致没有一个candidate可以变成leader，这样会无限重复这个过程，为了防止这个过程，需要避免多个candidate同时选举，这样设置election timeout为在某个范围的随机数内，极大降低出现vote split的概率。<br/>

## log replication
做完lab2B，回来再写