---
layout:     post
title:      "Scaling Distributed Machine Learning with the Parameter Server"
subtitle:   "paper reading"
date:       2019-02-20 19:09:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---

<br>这篇论文探讨分布式机器学习系统的参数服务器设计，一作是李沐大神，这应该也是Tensorflow、MXNet等框架的参数保存的实现方式。系统将数据和运算负担分散在worker nodes，将全局参数保存在server nodes里。系统支持异步通信，不同程度的一致性保证和失败冗余机制。针对机器学习算法特有的性质（对一致性不敏感，数据量和参数量巨大）做出一定的trade off。<br/>

## 场景和挑战
<br>ps（parameter server）主要针对稀疏特征的机器学习问题，也就是说一个worker node所有的数据不会包括所有的特征权重，只需要一部分参数即可，这个特点就让ps系统能通过增加服务器数量扩展处理规模。在大规模机器学习问题，数据通信是个非常重要的瓶颈，一般分布式系统通过key，value pair来传输数据，这在ps系统太过低效，机器学习一般采用vector、matrix或者tensor存储数据，所以ps也采用这种方式传输数据。另外一个问题就是节点失败处理方式，ps就像其他分布式系统，备份机制，热恢复机制都被加入其中。关于机器学习背景知识我就不介绍了，总结就是大规模，循环式处理，一致性不敏感。<br/>

## 架构
<br>系统首先将数据分配到每个work node里，然后将所需要的参数从server node拉过来，每个node并行求gradient，然后将计算完成的gradient push到server端，server端进行整合，然后worker node再将整完成的参数拉过来，这样一直循环直至收敛。大致流程就是这样，在大型机器学习系统里，ps可以同时负责不止一个机器学习任务，所以可以有多个worker group，每个group负责一个application运行，每个group（worker和server端）都有一个管理节点，负责任务分配，失败重启，节点加入等任务。这其中有几个问题，数据存储格式，数据传输格式，server端功能扩展，执行效率，一致性保证等问题。<br/>

<br>关于存储方式，就像上文中提到的那样如果采取简单的key，value pair方式，key是feature ID，value是float数字，就太浪费空间了，ps采取类vector，matrix方式，如果一段参数里有不需要的参数，采取补0操作，这样不仅节省空间，而且适合为矩阵运算优化。关于传输方式，ps采用range方式，每次传输连续的key数据，server端和worker端采取push和pull的方式进行通信，w.push(R,dest)代表将w参数里的R（key range）推送到目标节点，pull同理，这样的处理方式能够一下读取一段数据，提高通信效率。<br/>

<br>在机器学习算法里，server端一般负责整合梯度信息，在ps系统里允许用户自己定义函数在server node运行，因为server端的数据一般是最新最全的，可以进行进一步处理，如加入正则化的loss function导数处理或者其它函数处理。ps系统实现异步处理数据，因为机器学习算法天生对数据一致性不敏感（不要求每个循环进行都等待上个循环参数更新），当caller下发任务之后，可以直接进行之后的计算，当callee给caller回复之后，caller认为任务结束。在机器学习里，不同循环的计算可以是异步的，比如iter 10和iter 11可以同时进行，但是可以设置iter 12需要在iter10、11之后，这样保证执行效率也保证一定的一致性保证。ps系统采取bound delay一致性保证，它允许一定程度延迟，比如设置t=1代表第k个循环只需要k-2个循环完成即可，t=0就是顺序一致性保证，每个循环必须等到上个循环完成，t=∞代表所有循环可以同时开始。ps系统还支持用户定义过滤函数，可以用来过滤不重要的参数更新。<br/>

<br>在具体实现时候，server端采取一致性hash方式存储数据（DHT），因为在执行过程中需要记录数据时间戳，ps采用vector clock方式，为每个range数据标记同一个时间戳，这大大节省为每个pair打时间戳成本。节点之间依靠message传递，每个message包括list of (key, value) in range R和附带的时间戳。运行的时候，每个message的key list很可能是保持不变的，所以传输的时候节点可以cache key list，只用传输hash of list，同时value因为特征稀疏和value filter所以包括许多zero，所以只用穿越传输non-zero值，论文说这两种方式可以同时使用，我不明白那如何对应key，value值呢（每一步nonzero值可能不同）。<br/>

<br>ps系统采取DHT存储参数，每个节点存储k个逆时针邻居节点数据备份，为了节省网络流量和计算能力，server端整合过程只由mater完成，完成之后再将结果传输给slave节点。关于server端节点更改和失败处理，节点管理器会将key range分配给新node，这时候新节点会从其它节点获取属于range的数据，原始节点的range需要split，然后删除多余的数据，这时候收到的消息转发给新节点，并拒绝接收消息。worker端节点变更比较简单，新节点获取一批训练数据，然后从server端服务器获取所需参数，task scheduler广播信息，其他节点删除重复训练数据，由于损失一批数据对于足够大规模的数据没有太大的影响，可以选择不进行失败恢复,甚至主动删除一些执行缓慢的worker node。<br/>

## 问题
<br>ps系统针对稀疏特征有着很好的表现，但是对于NN网络或者其它稠密特征算法，通过增加服务器扩展参数量应该不会有很好的表现，只能保证数据量分散，无法保证参数量分散。<br/>
