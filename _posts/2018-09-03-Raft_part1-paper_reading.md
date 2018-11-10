---
layout:     post
title:      "In Search of an Understandable Consensus Algorithm (Extended Version)"
subtitle:   "paper reading"
date:       2018-09-03 15:57:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---

Raft是分布式系统中的共识算法，属于分布式系统里的基础，通过共识算法才能保证服务器失败容忍，即只要majority服务器还存活，系统就可以一直运行下去。Raft属于Paxo针对理解性的改良版，曾经我花了一上午时间看Paxo协议，结果只记得将军、间谍之类的东西了（误，这篇Raft写得更为详尽和清晰。由于Raft涉及太多细节性内容，在一篇博文很难解释清楚，所以本篇文章只将论文框架写出，具体的内容感兴趣的小伙伴可以直接读这篇论文为好。论文主要将raft分为leader election、log replication和safety三部分阐述，我也就分这几部分开始说。

## raft basics
<br>raft将在这个系统内的服务器分为三个角色：leader、candidate and follower。一个系统只有一个leader，正常情况其它服务器应该都是follower，在follower超时，需要竞选leader时候变成candidate。follower只能被动地接受leader的信息，leader负责处理client全部的请求。raft将系统运行时间分为任意长度的term，每个term以一次选举为开端（term被视作raft里的逻辑时钟，检测过时信息）。关于term检测和服务器三种状态的变换具体见论文的Figure 4，我就不赘述了。实际上所有的细节全部在Figure 2这张图里，理解raft务必仔细研究这张图。

## leader election
<br>leader通过周期性地心跳机制防止follower变成candidate，从而保住自己leader的位置。但是如果follower超时（也就是在某个时间段既没有收到leader的心跳也没授予其它candidate vote），变成candidate，term加一，准备选举，对所有其它request vote请求，这种情况只有三个结果：1.收到majority vote，变成leader 2.其它服务器变成leader，自己降格为follower 3.计时器过了，没有服务器变成leader。现在分别阐述每种情况发送的条件和后果，1 在相同的term收到majority的vote然后变成leader，之后直接发送heartbeat宣示主权。2 本candidate在等待vote时候，来了其它leader的heartbeat，自己就变为follower 3 多个follower同时变成candidate，开始选举，vote被分为多块，导致没有一个candidate可以变成leader，这样会无限重复这个过程，为了防止这个过程，需要避免多个candidate同时选举，这样设置election timeout为在某个范围的随机数内，极大降低出现vote split的概率。<br/>
<br>这里简单讲一下candidate如何获得其它副本的vote。第一个条件就是判断副本是否在当前term给出vote，如果没有，再判断candidate的log是否比副本的更uptodate，uptodate的判断条件：1.最后的entry谁的term更大2.如果term相同，就判断谁的log更长。<br/>

## log replication
<br>在做lab 2B的时候，我开始以为log replication遵循地是这个流程，leader从client接受到指令，写到log里，开始对所有follower和candidate发送commit请求，等到接受到majority之后，leader将log apply到自己状态里，然后对所有副本进行apply请求。但是这套流程有几个很不好处理的点，第一个client如何处理log顺序，如果之前的log还没处理完，可以接受下一个处理吗？如何处理？第二个论文要求如何log commit apply失败，如何leader没有失败，需要无限进行，直到成功，如何做到？第三个接收到majority之后leader自然可以apply log，但是怎么发送给副本，特别是指定已经完成commit的副本，而且如何知道结束？后来我才发现这套逻辑是错误的，也不能说是错误，只是在实现的时候，不能这么实现，论文只是为了方便描述，才解释这一套虚拟流程。实现时候，append log是和heartbeat一起实现的，也就是说每当leader发送heartbeat时候，通过nextIndex识别每个副本下一个要发送的log位置，将leader从那个位置的log全部发过去，而这个动作是按照心跳机制的周期性的，从而成功解决之前提出的第一个和第二个问题，而每当leader收到一个副本的commit成功之后，leader就开始判定commitIndex是否应该更新，如果更新的话，apply log，并将这个commitIndex通过下一个append rpc传给副本里面，让他们也能够定期apply log。而client发送指令给leader时候，只是增加leader的log，让下一次leader发送心跳时候识别应该发送更多log就可以了。<br/>
<br>在这里有几个raft规定的要求需要实现。第一个leader append only，意思leader不可以覆写，删除自己的log只能append，这个要求在实现已经满足，leader负责接受client的指令，打进log。第二个log matching--如果两台不同服务器的log有着相同index和term的entry，那么在这之前的entry应该全部相同。在实现的时候，因为leader只会在指定term一个位置append一个entry，这样其它副本出现同样的index和term的entry时候，必然是leader commit过去的，必然含有相同的command。关于log matching，每次leader发送append rpc都会发送副本应该append log前一个位置的index和term，保证它们和副本现在最后一个entry相同，如果不相符就拒绝append rpc，leader将nextIndex提前，重新发送，知道找到符合条件的位置。这样的话，按照递推，最开始都是空状态肯定符合log matching，然后每次添加log都检查index和term，这样保证每当返回成功的话，副本和leader直至新entries都是一样的。<br/>
<br>在正常操作下，副本和leader的log会一直保持一致的，但是如果leader crash，再上线后，就没有全部的log了，这时候新leader就需要对副本进行覆写和删除，和上一节所说的一样，通过leader的nextIndex找到副本和leader一致的最后一个entry，然后把leader之后的log全部复制进去。<br/>
<br>第三个要求是leader completeness，如果一个entry在一个term被commit，那么在之后所有高于这个term的leader都应该有这个entry，具体证明比较复杂，感兴趣的同学可以直接看论文5.4.3节的证明，大致意思是commit要求majority，candidate变成leader也要求majority，这其中必然至少有一个server重复，所以commit log必然一直在leader里面继承。第四个要求是state machine safety，意思是如果一个服务器在指定index apply一个entry，那么其它server必然在指定index apply相同的entry。因为apply entry之前必须要commit，这就确定之后的leader必然有这个entry，保证即使在之后的term apply，在相同的位置也是相同的值。<br/>
<br>还有一个注意的点，每次leader commit entry时候，不会commit之前term的entry，之后commit当前的entry。因为这和判断uptodate会产生矛盾，会出现commit log消失在之后的leader的情况，，详见论文Figure 8.<br/>

## persist
<br>lab 2C是最简单的了，在raft协议运行期间对server状态进行存储，然后在它们失效后从磁盘里恢复这些状态。raft论文给出需要存储currentTerm, voteFor, log,具体实现的时候，每个method如果对rf server的这三个状态进行修改的话，都需要重新写入，在启动server，也就是make的时候，读入最新的状态就行了，很简单的逻辑。<br/>