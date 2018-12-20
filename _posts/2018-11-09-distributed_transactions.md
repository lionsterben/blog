---
layout:     post
title:      "Distributed Transactions"
subtitle:   "lecture study"
date:       2018-11-09 17:37:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - lecture study
---

## Transaction atomicity
<br>本篇文章主要关注的是分布式事务的处理，首先介绍介绍一下事务的概念。事务具有原子性、可序列化和持久性，原子性是指这个事务要么全部完成要么不处理，关键是在执行过程中失败，该怎么处理，数据库是undo（貌似是的），在分布式系统中间结果写在缓存里，最后再commit，如果协调者发出commit命令，即使worker失败，整个系统也会等待它恢复返回commit成功信息，在commit之前如果出现问题，由协调者直接发送abort命令给所有worker。可序列化是指并发的事务结果需要是顺序执行这些事务结果之一（注意这里保证只要是其中之一即可，并不限制事务的顺序），也就是在上一层调用者面前，这些事务虽然可能是交叉进行但是在调用者面前看起来就像顺序执行的一样。持久性保证已经commit的数据即使失败也要保证存活。这些给出很好的语义给予上层的调用者。接下来两节介绍并发控制和原子性保证<br/>

## Simple locking and two phase locking
<br>我们主要关心分布式系统并发事务可序列化的保证，主要分为消极式控制和积极式控制，消极式控制是先获取事务进行需要的lock，如果冲突就等待。而积极式控制则是直接处理，不需要锁，等到commit时候检测是否冲突，如果冲突只能abort重来。可以看出如果事务冲突不明显的话，积极式处理效率更高，这篇主要是依靠锁的处理即消极式处理，积极式处理将在Farm这篇博文中介绍，敬请期待。<br/>
<br>最简单的lock控制是每个事务先获取事务进行的所有锁，我们称获得所有锁的这个时间点为mark point，之后再进行处理。假设只有两个事务，那么这两个事务可以看成按照到达mark point的order处理的，但这种方式丧失并发处理的机会，大大浪费计算资源。所以引入2pl，在事务进行时候获取锁，也就是说边进行边获取锁，但是锁一旦获取只有等到最后commit或者abort时候才释放，这时候事务的顺序也是达到获取所有锁的mark point决定，这里大大提高了事务之间的并行程度。这里举个例子，T1：a,b,d T2：p,q,d，在simple locking里T1首先获取a,b,d的lock，这时候T2获取p和q的锁之后是阻塞的，只有等到T1完成之后释放d的锁之后才能进行。而在2pl里，T1和T2的前两个执行过程是可以并发执行的，提升处理速度。这里介绍一下在abort和recovery情况下的处理措施，abort比较简单，只需要保持修改过的数据的锁，将修改过的数据还原即可，完成之后才能释放这些锁。crash就比较麻烦了，恢复的时候需要log的协助，假设锁是在volatile storage，如内存，crash之后锁的信息直接消失，但是可以看出这些锁与其它事务是非重叠的，只要这时候暂停其它事务的进行，先根据undo 反向执行这些log，就可以保证serial order。课件里说的是在恢复过程里，阻止新事务的开始即可，但是执行中的事务获取恢复中数据的锁怎么办？不会冲突吗？<br/>

## Two phase commit
<br>这节介绍一下分布式事务里保证原子性的操作，因为在分布式系统里数据是分布在不同机器里的，所以事务需要被分解成几个子任务给不同的机器。首先介绍一下2pc的流程，首先有一个coordinator（协调者TC），发送RPC将任务分配给不同的worker，worker将结果暂存，不要提交。TC将事务分解发送完成后，就可以发送prepare message给worker，询问它们是否准备好commit，如果worker回复yes，则进入prepared状态，如果所有worker回复yes，则进入commit阶段，否则进行abort阶段，之后TC发送commit/abort信息给所有的worker，commit将暂存信息写到永久状态里，然后释放锁，abort直接释放暂存信息，释放锁。<br/>
<br>在整个过程里，存在多处可能失败的地方，先说worker的失败，如果worker在prepare回复yes之后失败，worker一定要记得自己的暂存信息，因为TC可能会进入commit阶段，其它的worker状态会永久改变，该worker在reboot后，TC会让它再次commit，所以worker一定要把暂存信息刷到磁盘里。如果worker等待TC的prepare信息，这时候crash了或者等待TC的prepare请求超时了，这时候TC不会发送commit信息，可以单方面abort，在reboot之后发送no信息。如果TC没有收到其中任一worker的prepare回复，可以直接认为abort，释放其它worker的锁。所有worker只要回复yes就必须等待TC的decision。如果TC crash，TC如果发送commit信息它必须记得，因为一部分worker可能没有接收到，在restart之后需要继续处理，所以commit信息必须刷到磁盘里。<br/>
<br>可以看到2pc虽然解决分布式事务的原子性问题，但是存在很多问题，比如多轮信息传输，缓慢的磁盘读写，lock从rpc调用到最后commit或者abort才能释放，TC crash会导致worker无休止的等待（hold locks）直到TC恢复。<br/>