---
layout:     post
title:      "Convolutional Sequence to Sequence Learning"
subtitle:   "paper reading"
date:       2018-10-04 11:08:00
author:     "Dawei"
header-img: img/nlp.jpg
tags:
    - lecture study
---
## 概述
<br>本篇论文是Facebook AI Research使用纯CNN网络来进行机器翻译，很棒的思路，同时还diss了一下LSTM的思路（我觉得是冲着Google去的，结果Google过一段时间就放出attention is all you need，哈哈），关于传统RNN的缺陷我在这篇[博客](https://lionsterben.github.io/lionsterben/2018/09/17/attention_is_all_you_need/)里提过了，就不多说了。主体框架也是包括encoder和decoder两部分构成，但是两部分都是由相同的layer（CNN+non-linearity）stack而成，每层decoder layer都应用attention机制。<br/>

## Convolutional Architecture
<br>这里就简单说下我看得时候感到疑惑的地方，具体的细节还是看论文吧。假设k是卷积一维window size，d是embedding dimension。输入的时候和attention is all you need那篇论文一样，需要加上位置信息，论文采取直接相加的方式。说下layer组成，显示一层卷积，对于一个windows size的卷积，论文设置的$2*d$个filter，所以出来的结果是$R^{2d}$的结果，然后进入gate linear units(GLU)进行非线性化，将卷积输出的$R^{2d}$分为两半，都是d维，假设为A,B，对B采取sigmoid计算A的权重（计算和当初decoder需要的output的相关程度），之后和A进行element-wise即可，最终输出$R^d$，这是一个window size输出的值。在encoder里，论文保证每层layer输出的output和原始input长度相同，所以要对input进行padding，这样stack几层layer既可以作为encoder了，哦对了，论文还使用了resnet，每层layer的结果都会最后加一下这层layer的输入。<br/>
<br>这里着重说下decoder，也是我疑惑颇重的地方，开始我以为和encoder一样先stack几层layer，最后做attention，但是怎么也说不通，还有我以为decoder也需要output length和input length相同，还有就是论文给出的框架图是training图，并行化处理的，我开始没看懂。先说下decoder的流程吧，每一层layer和encoder layer前面相同，都是卷积和GLU，为了简单起见，我就以decoder只有一层decoder layer解释了，decoder的输入不用全部已经output的结果，只用在当前windows size（卷积窗口），论文的图里就是之前的3个预测之后的一个，当然你可以stack几层获得更大的视野。接下来不以论文里并行化处理的解释，以单个example解释，经过一层layer输出的和encoder里的一样$R^d$，这时候如果是多层就需要resnet处理一下。论文在将这时候的decoder state和decoder上一步输出的值组合在一起，然后就和之前介绍的attention机制一样，计算权值，对encoder状态加权平均（论文发现加权平均的时候加上原始input embedding效果会变好）得到context vector，直接和这层的hidden state相加，得到最终的hidden state，作为下一层的输入，如果是最后一层的话，直接矩阵相乘softmax就可以输出概率了。注意每一层都需要attention机制，每一层的输出是逐渐变少的，比如decoder的输入是25个word($R^{25*d}$)，假设卷积窗口大小是5，到第6层输出的时候只会剩下$R^d$了。<br/>

## 几个小点
<br>论文指出训练时候一个句子里多个位置可以同时训练，因为不像RNN里面依赖输出的前一个hidden state。论文的框架解决重复翻译的问题，因为每一层输入获得上一层的hidden state包括上一层的context vector，知道卷积窗口的目标word对哪些source word权重高，就会避开重复翻译。论文还给出几个trick，关于参数初始化和normalization的技巧，而且还给出了数学证明，有兴趣的同学可以看下。<br/>