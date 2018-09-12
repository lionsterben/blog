---
layout:     post
title:      "CS224N neural machine translation and models with attention"
subtitle:   "lecture study"
date:       2018-09-07 17:21:00
author:     "Dawei"
header-img: img/nlp.jpg
tags:
    - 技术随想
---

本文简单说一下NMT的机理和各种fancy model、trick。主要分析Sequence to Sequence Learning
with Neural Networks、Neural Machine Translation By Jointly Learning to Align and Translate and Effective Approaches to Attention-based Neural Machine Translation这三篇论文。

## Sequence to Sequence Learning with Neural Networks
<br>作为我看的第一篇NMT论文，这篇论文也算是NMT的早期具有权威性的论文了。论文主要提出两个LSTM网络进行translate，一个是encoder，最后输出source sentence的represention，另一个是decoder，对source sentence的represention进行decode，从而输出target sentence。相比于传统RNN不善于捕捉长距离联系，论文选用LSTM作为网络结构，实验发现在长句子，LSTM的表现的确优于一般RNN。另外论文发现多层LSTM的表现比shallow LSTM的表现好很多。最后一个trick就是将source sentence的单词顺序颠倒过来，能够极大提高性能。<br/>

## Neural Machine Translation By Jointly Learning to Align and Translate
<br>这篇论文发现传统模型中encoder输出的fixed-length vector已经成为瓶颈（也就是说这个vector不足以存储那么多信息，encoder导致开始信息丢失？或者说对于decoder太难解析出来了），论文引入attention机制，为decoder每步输出单词时，通过align机制给予输出单词对应的source sentence信息。<br/>
<br>论文采用双向LSTM结构作为encoder部分，，将每步的hidden state输出出来，作为decoder的信息输入。decoder部分，这一步的结果由上一步输出的结果，本步的hidden state和context vector决定，重点在于context vector，其它和传统lstm无异。decoder本步的hidden state由上一步的hidden state、输出加上本步的context vector决定。context vector是encoder输出每步hidden state的加权平均，关于这个加权的参数由decoder上一步的hidden state和每个encoder的hidden state决定，这些参数决定对于decoder本步哪些encoder的hidden state重要性，间接给出decoder的word由那些source word决定（align mechanism）。论文在最后给出详细的公式和细节，我就不赘述了。<br/>

## Effective Approaches to Attention-based Neural Machine Translation
<br>论文对于attention机制进行改进，并将attention分为global attention和local attention，上一篇论文就属于global attention，对source word每一个都计算权重，而local是指先预测到应该对应source某个单词i，然后利用超参数d，取[i-d,i+d]内的单词进行attention。<br/>
<br>论文里提出的模型和第二篇的还不太一样，第二篇将encoder和decoder完全分离，具体见上一段，而本文延续第一篇的结构，也就是原始的encoder和decoder还是连在一起的，decoder还是会输出原始h_t，但是输出的y_t的decoder是由context vector和原始h_t计算得到，context vector的权重也是由h_t和encoder的每个hidden state决定，所以每个context vector计算都是独立的。在local方面，决定source word i有两种方法，第一个十分粗暴，单调对齐，每一个对每一个，权重公式用原先的。第二个需要预测，具体公式见论文公式9，然后权重公式增加标准分布乘项，大概意思就是离预测的word值越近，权值越大。<br/>
<br>论文与第二篇论文相比，计算简化，由原h_t和context vector直接得到最终hidden state，第二篇论文是先得到decoder上一步hidden state，y_t-1，然后context vector，最后才是hidden state。并且在计算attention权值时候选取的公式论文进行比较，简单将h_t和hidden state concat在一起计算，无法使它们进行交互（类似上一篇论文的用法），论文提出h_t*W*h_s能够充分捕获非线性信息。但是论文也探究提出的模型在decoder时无法获得之前步骤align的source word，提出input-feeding approach，也就是将final decoder每一步输出的最后hidden state作为下一步原始decoder的输入，获取align信息，具体见论文图4。<br/>
<br>Deep Learning的论文普遍都很简洁，往往只是提供fancy model或者几个trick，相比于一些大型工程型论文，好读很多。<br/>
