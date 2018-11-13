---
layout:     post
title:      "CS224N Coreference Resolution"
subtitle:   "lecture study"
date:       2018-10-14 09:58:00
author:     "Dawei"
header-img: img/nlp.jpg
tags:
    - lecture study
---
<br>在语言里，经常会出现多个名词指代实际世界里同一个实体，将指向同一个实体的名词筛选出来就是Coreference Resolution task。这在分析语句，语义分析非常有用，这篇博客就介绍一下在深度学习时代几种出色的模型，mention-pair、mention-rank，reinforcement learning。<br/>
<br>Mention Pair将共指任务变成两个mention之间的连接关系，也就是说每两个mention都需要判断一下是不是指向同一个实体，如果是的话，就将它们连接起来，最终同属于一个cluster的mention会形成一张网。如果用这种处理的话，就将这个问题变成binary classification task，将两个mention的特征和额外特征喂进去，加上简单的神经网络就可以判断了。<br/>
<br>Mention Rank则将共指任务表现为链式结构，也就说对一个mention，对之前出现的mention和表示该mention不指向任一mention的new标记通过神经网络输出一个rank，最后选择rank最大的mention表示为它的共指mention。score的表示和上一节的二分类任务很像，不过在处理new标记的时候，只使用该mention的特征来计算score，而其它的则变成该mention的特征和两个mention特征的结合。关于mention的特征这里提一下，单个mention有mention的中心词，第一个单词，最后一个单词，之前的mention，之后的mention，NER tags等等，而两个mention结合的特征首先是两个mention的基础特征，文档类型，是否同一个speaker，它们的head是否match等等。<br/>
<br>这个model的loss也可以说一下，任务的结果会出现三种错误，wrong link：指向错误的mention，false new：模型错误预测new，false link：应该指向new，错误指向任一mention，这三种错误的严重性是不一样的。所以在loss里面的权重也应该不一样。loss是max-margin loss，希望指向错误的mention的score尽可能小，这是通过尽可能减小score和cost（错误代价）相乘的最大值得到的，正确的cost是0，其它的错误则是指定的。<br/>
<br>共指问题也可以通过强化学习解决，我们使用B3指标来表示性能，我们可以直接把这个指标当作对一个样例添加link最终的奖励，通过最大化这个奖励来寻找合适的策略函数。还有一种很有趣的用法，之前提到错误的cost可以通过强化学习来确定，对于一个action即一个link，我们可以固定其它action，对于这个action我们可以改变它寻找最大的reward，相减就可以得到它的代价。<br/>