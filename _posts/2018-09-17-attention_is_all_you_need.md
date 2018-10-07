---
layout:     post
title:      "Attention Is All You Need"
subtitle:   "paper reading"
date:       2018-09-17 22:06:00
author:     "Dawei"
header-img: img/nlp.jpg
tags:
    - paper reading
---

## 概述
<br>论文主要提出用于机器翻译新的模型。机器翻译分为encode和decode部分，encoder对source sentence进行编码，在RNN一般是最后输出的hidden state。decode是对encode部分输出的hidden state进行串行解码，后来引入attention机制，对encode中每个hidden state都引入计算。CNN用于机器翻译我在另一篇博客里介绍吧。可以看出RNN虽然符合人类的直觉，但是天生不符合GPU并行计算的特点，所以训练速度很慢。谷歌这篇论文对encode和decode部分，直接放弃原始的rnn网络，使用self attention进行编码和解码，非常interesting。<br/>

## Attention
<br>首先介绍下论文的attention结构。广义上的attention需要query Q、key K、value V，query和key确定value的权重，权重和value相乘就得到最后结果，在machine translation里，encoder的query、key、value全部是上一层的输入，decoder的query来自上一层的输入，key、value来自encoder的输出，有点像原始attention机制了。再举一个例子，在问答系统里，query是文章向量，key，value就是问题的向量。<br/>
<br>论文先介绍简单的dot-product attention，就是很简单的矩阵乘法，首先是Q与K相乘，经过scale（调整分布，防止softmax偏向两端），之后是可选的mask选项（用在decoder，屏蔽掉当前状态不需要的信息），之后用softmax输出权重和V相乘，就结束了。论文之后提出multi-head attention，对V、K、Q分别做线性变换生成h个不同的V、K、Q组合，喂入dot-product attention，生成h个结果，之后将它们concatenate在一起，最后做个线性变换输出。<br/>

## Attention机制作用
<br>论文在encode端通过简单的矩阵乘法，获取source sentence两两位置的关系，很好的对sentence编码，复杂度是O($n^2*d$)（n是sentence长度，d是word vector维度），对比RNN encode的复杂度O($n*d^2$)，一般句子长度比维度小很多，而且self-attention是一下子计算完成，很容易并行化，相比RNN计算量又大，还要串行计算，就快很多了。另一点就是input和output依赖距离关系，self-attention可以将任意两个位置的input和output联系起来，原始RNN则需要跨越整个句子。但是需要注意的是到目前为止，该模型完全抛弃了语序信息，也就是说随意打乱单词顺序，求出来的结果是一样的，所以论文指出需要引入位置信息--position encoding，论文直接给公式，利用正弦、余弦公式对位置进行encoding，说是可以反应相对位置，我没太看懂囧。<br/>

## Model Architecture
<br>论文给出的框架，我简单介绍下，encoder端将input（句子）先做embedding，与position embedding相加，引入位置信息，喂入block，block由两个layer组成，第一个layer就是multi-head attention加上residual和norm机制，第二个layer是简单的全连接层加上residual和norm机制。论文采用6个block连在一起。decoder端和encoder类似，输入当前预测位置之前的output，embedding，plus position embedding，因为decoder需要屏蔽掉多余信息，所以第一个layer采用mask multi-head attention，第二个layer的输入中Q是decoder上一个layer的输入，K,V来自encoder端的结果。之后的和encoder相同，也采用6个block连接。论文没有清晰地给出几个block stack在一起如何处理中间输入的，从[这篇博客](https://mchromiak.github.io/articles/2017/Sep/12/Transformer-Attention-is-all-you-need/#.W7WTqWhLi70),我大致描述一下，encoder端没什么好说的，一层一层递进就可以了，在decoder端，每层block中第二个layer输入应该是encoder端最后输出的结果，而不是每一层对应的结果。decoder在训练的时候可以并行，因为有ground truth，而且每一个预测的结果也不会依赖前一个hidden state，但是在预测时候，还是需要一步一步生成单词。<br/>
<br>先大致写到这，感觉还是有些地方没有弄通，等有空实现这个算法的时候，遇到坑再回来填吧。<br/>
