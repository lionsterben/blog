---
layout:     post
title:      "ZooKeeper: Wait-free coordination for Internet-scale systems"
subtitle:   "paper reading"
date:       2018-10-18 15:40:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---

<br>这篇论文比较硬核，我已经看了三遍了，还是有些细节没有弄清楚。我先把我所理解的东西写出来吧，以便以后要用到时候回忆和勘误。zookeeper作为分布式系统中协调者的角色出现，它提供几个基础的api，供使用者自己搭建系统中协调者的业务。Zookeeper强调wait-free属性，带来极高的性能提升相比于blocking实现。为了实现协调者功能，Zookeeper保证了client FIFO order和all write requests linearizble。整个Zookeeper是由层次文件系统组织起来的，提供的API也是基于文件系统的。<br/>

# Zookeeper Service
<br>关于zookeeper提供的服务我就不展开了，大概说一下。整体zookeeper是由多个server组成，其中包括一个leader，其它都是follower，和raft类似。每个需要zookeeper服务都在自己的服务器上运行zookeeper client，通过client和zookeeper server连接，获取zookeeper服务。client可以和任一zookeeper server连接，read request可以直接通过这个server获得信息，但是write request等要改变zookeeper state的操作，需要转交给leader，由leader通过zab（类raft）操作。Zookeeper将application需要在上面存储的信息抽象为znode，以层次文件系统的方式存储，获取的时候也是通过目录路径的形式指定znode。<br/>
<br>znode包括两种，一种是正常节点，由client显式创造和删除，另一种是临时节点，区别就是当创造这个节点的client session失效后，系统会自动删除，而且它不能创造子节点。create可以设置sequential flag，作用在于node的名字最后增加一个递增的值。getData操作可以设置watch标志，在get的这个znode之后再改变的时候zookeeper可以通知client。<br/>
<br>介绍一下session概念，和普通的session很像，client和Zookeeper交互就通过session，session可以通过client主动关闭，zookeeper也会通过Hearbeat机制探测client是否失败来关闭。session机制可以保证client透明地改变连接的server。<br/>
<br>client API。create znode，delete，exists，getData，setData，getChildren和sync(PATH)。这里主要说下sync操作，这是为了保证read操作读到最新的数据，是连接的server直到server update到这个操作提交的时刻所有的操作都完成之后再返回read结果。<br/>

# Zookeeper guarantees
<br>好的，到重头戏了，这里大家带着批判的眼光来看，因为我也不知道我的理解是否正确。Zookeeper保证writes的linearizable和单个client的FIFO order。因为Zookeeper允许client进行异步操作，所以论文里说linearizability和之前的不一样，这里我不太懂，我所搜集到的资料显示linearizability是针对单个对象，所有操作按照实际顺序执行，Herlihy定义的linearizability针对的是client每个时间只能有一个operation，但是zookeeper允许client异步操作，也就是在同一时刻可能会有多个操作等待处理，这里zookeeper通过pipeline架构保证单个client FIFO order，之后论文的几句话我没有读懂。<br/>
<br>论文举了一个例子来说明这些保证的作用。当使用zookeeper的系统改变leader，从而要改变zookeeper很多配置文件，这时候需要两个保证：1.当new leader开始改变配置时候，我们不希望其它worker读取配置 2.如果new leader在更改配置时候宕掉了，我们不希望worker使用不完全的配置。那么Zookeeper怎么保证呢？new leader在配置文件更改全部完成后，写一个ready znode，别的worker只有在看到这个znode才可以读取。这一点由new leader所在client的FIFO order保证异步操作中create ready znode在其它更改配置操作最后，linearizability保证zookeeper全局write操作按照实际order执行，所以在worker可以根据ready znode获得配置更改是否完成的信息。当然这个机制还有一点缺陷，这里我就不展开说了，感兴趣的小伙伴可以自己思考，而且解决方法已经在API里给出了哦，当然，也可以直接看论文2.3节。<br/>
<br>之后论文提到了一系列在zookeeper基础搭建的基础组件包括简单锁、读写锁和在一些使用zookeeper的应用，我这里就不展开说了。<br/>

# ZooKeeper Implementation
<br>硬核点又来了，当然这里大家也带着批判的眼光来看。这里先提一句wait-free是什么意思，我直接截取MIT课件的内容了。
>Q: What does wait-free mean?

A: The precise definition is as follows: A wait-free implementation of
a concurrent data object is one that guarantees that any process can
complete any operation in a finite number of steps, regardless of the
execution speeds of the other processes. 

The implementation of Zookeeper API is wait-free because requests return
to clients without waiting for other, slow clients or servers. Specifically,
when a write is processed, the server handling the client write returns as
soon as it receives the state change (§4.2). Likewise, clients' watches fire
as a znode is modified, and the server does *not* wait for the clients to
acknowledge that they've received the notification before returning to the
writing client.

Some of the primitives that client can implement with Zookeeper APIs are
wait-free too (e.g., group membership), but others are not (e.g., locks,
barrier).    
我的理解是保证每个操作在有限时间完成，不用关注别的操作，只要自己的operation完成就直接返回结果。<br/>
<br>这里说下write request执行过程，首先client发送request到leader，leader先把这这个操作变成幂等的transaction（执行多少次都会是同样的结果），leader通过计算当前state和这个操作来计算之后的state，把这个state更改作为动作之后执行（变成txn是幂等的）。txn在传给atomic broadcast，利用zab（类raft）协议进行txn广播备份，并且在失败之后，可能重发txn，保证幂等就是重复执行不会导致结果变化（和raft区别？）之后就是备份database，就是每个zookeeper内部状态，这里没什么好说的，平常执行包含在zab协议里了，其中snapshot提一下，它不用锁state来获取整体状态，对状态树进行DFS扫描，在扫描过程中，状态可能会改变，但是没关系，反正txn是幂等的，执行txn就可以了，我的理解是在flush一个tree到磁盘之后，就可以把扫描开始之前的txn全部抛弃掉了。<br/>
<br>最后说一下，client和server的交互，read和write前面都提到的一些。这里说一下read，因为write有lineariablility保证，但是read可能会读到stale data，这里有一个zxid指标，我的理解是每次read之后，server都会但会一个zxid给client来表示这个数据的有效期，但是我的理解是这只能保证单个client的partical order，也就是保证这个client的顺序，当我们想要最新的数据，可以利用sync操作，这必然返回当前时间最新的数据，但是需要更多的等待时间。论文最后一段感觉zxid这个东西又是全局性的，应该是client提出write操作时候也会返回zxid数据，server自身也会保存自己的zxid，当client需要转移的时候，会比较自身的zxid和server的zxid，只有server的zxid赶上client的zxid才会成功对接。<br/>
<br>关于Zookeeper先暂时写到这些，之后有新的想法或者更正再加吧，写死我了。<br/>


