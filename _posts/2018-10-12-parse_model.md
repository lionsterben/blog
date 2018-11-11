---
layout:     post
title:      "Natural Language Parsing Model"
subtitle:   "project thinking"
date:       2018-10-12 09:40:00
author:     "Dawei"
header-img: img/starry_night_sky.jpg
tags:
    - project thinking
---
<br>句法分析在NLP是基础研究点了，在一些下游任务比如情感分析、信息抽取都具有较大的作用。句法分析通过给出句子中的句法结构或者依存关系，是指给出主谓宾等成分和单词之间的修饰关系。举个栗子，我爱你和你爱我这两句话，如果在原始的词袋模型里，表达出来的意义是完全一样的，但是结合句法就完全不同了。在CS224N里介绍了Dependency Parsing，但是在之后的TreeLSTM也提到了Constituency Parsing。我就简单介绍一下这两种parsing方法。<br/>

## Constituency Parsing
<br>首先提一下这种方法，这个方法和我们平时认知类似，从上往下看，可以将整个句子分解为多个短语，而每个短语则是由更小的短语组成直至单词，从下往上看则是每个单词组成短语，逐渐扩充到达整个句子。我记得看过一篇论文将语义分析和parsing结合在一起，计算相邻的两个节点组合的score，取其中最大的两个结合在一起，这里面的hidden state还可以做语义分析。这样从单个词语开始的节点逐渐组合变成拥有root节点的树，就完成整个parse过程了。这样每次组合都会生成更大的短语，并且短语组合的顺序是有意义的，代表修饰主被动关系。<br/>

## Dependency Parsing
<br>Dependency Parsing和Constituency Parsing不同，它是直接给出单词和单词之间的依赖关系，或者说标注关系。每个单词可以被多个单词修饰，而每个单词只可以修饰一个单词，通过不断这样处理，将所有单词之间的关系确定直到只剩下root就可以结束了。CS224N介绍一种使用神经网络的贪心Dependency Parsing Model，先设置buffer和stack，初始时，将整个句子塞进去，网络的输出是left_arc, right_arc, shift这三个动作，当然arc包括不同种类的标记，比如主谓关系、动宾关系等，每次left_arc或者right_arc将修饰词从stack里面删除，因为一个单词只能修饰一个单词，shift则将buffer里的头部单词添加到stack里面。那么如何决定这个动作呢，那就要依靠神经网络喽，输入是stack和buffer里面头三个单词和buffer的头两个单词各种已经决定的修饰词，另外加上这些单词的词性（POS tags）和那些修饰词的arc label，这样全部喂进去，由网络得出下一步该采取的动作。效果非常不错，保证速度的前提下，达到92%的准确率。<br/>
<br>在Tree-LSTM模型里，Dependency Parsing Tree不需要考虑子节点的顺序，毕竟作为修饰词是没有去别的，Constituency Parsing就不是单词之间的修饰了，单词之间组合形成短语，这时候单词顺序就有意义了，这一点在选取Tree-LSTM时候需要注意一下。<br/>