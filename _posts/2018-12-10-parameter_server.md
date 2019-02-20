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

<br>这篇论文探讨分布式机器学习系统的参数服务器设计，一作是李沐大神，这应该也是Tensorflow、MXnet等框架的参数保存的实现方式。系统将数据和运算负担分散在worker nodes，将全局参数保存在server nodes里。系统支持异步通信，不同程度的一致性保证和失败冗余机制。针对机器学习算法特有的性质（对一致性不敏感，数据量和参数量巨大）做出一定的trade off。<br/>

## 场景和挑战
<br>ps（parameter server）主要针对稀疏特征的机器学习问题，也就是说一个worker node所有的数据不会包括所有的特征权重，只需要一部分参数即可，这个特点就让ps系统能通过增加服务器数量扩展处理规模。在大规模机器学习问题，数据通信是个非常重要的瓶颈，一般分布式系统通过key，value pair来传输数据，这在ps系统太过低效，机器学习一般采用vector、matrix或者tensor存储数据，所以ps也采用这种方式传输数据。另外一个问题就是节点失败处理方式，ps就像其他分布式系统，备份机制，热恢复机制都被加入其中。关于机器学习背景知识我就不介绍了，总结就是大规模，循环式处理，一致性不敏感<br/>

## 架构
<br>系统首先将数据分配到每个work node里，然后将所需要的参数从server node拉过来，每个node并行求gradient，然后将计算完成的gradient push到server端，server端进行整合，然后worker node再将整完成的参数拉过来，这样一直循环直至收敛。大致流程就是这样，在大型机器学习系统里，ps可以同时负责不止一个机器学习任务，所以可以有多个worker group，每个group负责一个task运行，每个group（worker和server端）都有一个管理节点，负责任务分配，失败重启，节点加入等任务。这其中有几个问题，数据存储格式，数据传输格式，server端功能扩展，执行效率，一致性保证等问题。<br/>
