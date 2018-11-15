---
layout:     post
title:      "Reinforcement Learning and NLP"
subtitle:   "project thinking"
date:       2018-10-12 10:40:00
author:     "Dawei"
header-img: img/starry_night_sky.jpg
tags:
    - project thinking
---
<br>这篇博客简单对我现在所学习的强化学习知识做一个梳理。强化学习是个很大的话题，这里我只是介绍一下最基础的算法，可以看成对自己这方面知识的梳理。这里介绍下强化学习的基本组成部分，value iteration，policy iteration和DQN<br/>

## reinforcement learning introduction
<br>强化学习具有很强的通用性，这个概念很早就提出来了，随着深度学习的兴起，深度强化学习就逐渐流行，特别是Alpha-go的成功，它的项目负责人Davil Silver甚至说出了AI = DL + RL。但是天下没有免费的午餐，强化学习也有着数据量要求过大，需要精心设计的reward，训练难收敛等问题，但是毕竟取得如此大的成功，我认为这个方法也是现有人工智能领域最接近通用人工智能。强化学习的一般包括智能体agent，环境environment。agent通过观察environment获得当前state表示和给出的reward，agent会执行action对environment产生影响，environment会根据agent的action改变state和返回reward，目标就是在采取一系列的action之后获得的最终reward最多。state和action存在映射关系，agent会根据收到的state给出action，这也是我们需要学习的策略，问题就转换为寻找最优的策略来使最终reward最大。<br/>
<br>强化学习能够实现建立在这样一个前提下，未来只取决于当前，即遵从马尔可夫决策过程（MDP），状态之间的迁移可考虑当前状态即可。MDP可以用（S，A，P）表示，state，action，状态转移概率P（由state和action转移到下一个state的概率）。我们需要寻找的策略就是给定state输出action，可以转入下一个状态，周而复始，直到结束。关键是要使最终回报最大。强化学习的回报是引入折扣因子的每一步reward相加，越近的reward折扣因子越高也就是越重要。但是除非我们过程结束，我们无法获得之后每一个的reward，自然也无法获得state的回报，这里我们引入value function表回报的期望，即一个state的未来潜在价值。如何获得最大的reward呢？一种是优化策略即policy iteration，第二种是估计value function间接获得最优策略。<br/>
<br>接下来就是大名鼎鼎的bellman公式了，这是如何求解value function的关键所在，Bellman方程将计算当前状态的value function变为计算下一步状态的价值和当前的反馈reward有关。这表明value function可以通过迭代来计算。在这里我们可以计算在当前state采取某个action的价值，即Q(S,a)，如果固定策略之后，就可以通过迭代计算当前策略下每个Q(S,a)的价值，而我们需要求解的是最优策略，最优策略就是在所有策略里是Q(S,a)达到最大的策略，所以计算最优策略下的Q(S,a)就是当前的reward和下一步最优策略下最大action的Q值。<br/>

## Value iteration Policy iteration
<br>这里就可以引出value iteration和policy iteration，之后就简称为VI和PI，VI就是直接计算最优策略下的value值，不断迭代计算state下的value直至收敛。最后得到的是每个状态下的value值，策略就是计算不同action下获得最大value的action。PI，则是先固定策略，计算在这个策略每个状态的value，再根据得到的value值更新策略，这样不断收敛。注意哦，这两种方法都依赖状态转移概率才能计算而且得到的策略都是确定性的，也就是state对应唯一的action。<br/>
<br>Value iteration每一步迭代都需要对全部的状态和动作进行遍历，我们一般只能获得有限的样本，那么怎么才能处理呢。这里就可以引入Q learning了，在对每一个样本Q(S,a)进行更新的时候，不是直接将新的Q值赋给它，而是像梯度下降一样，将两个Q值的差值计算乘以learning rate，不断逼近最终Q值。这种方法需要存储所有状态和动作的组合Q值，这会产生维度灾难的后果，所以我们可以通过一个函数近似Q(S,a)，我们希望输入状态，然后输出各个action对应的Q值，这时候就可以通过一个神经网络来拟合这个函数，loss的定义就是这个更新的Q(S,a)值和上一次的Q(S,a)值的差距，我的理解是状态转移概率可以看成抽样样本（state,action,reward,state）的分布。这就是DQN，还是基于value iteration的方法，之后再加上深度学习和policy iteration的组合吧。<br/>
