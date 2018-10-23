---
layout:     post
title:      "CS224N Tree Recursive Neural Networks and Constituency Parsing"
subtitle:   "lecture study"
date:       2018-10-08 10:08:00
author:     "Dawei"
header-img: img/nlp.jpg
tags:
    - lecture study
---
<br>对于自然语言处理，之前介绍了RNN和CNN两种模型，RNN符合人类的习惯，序列化encode和decode模型，但是不适合GPU的并行化计算，CNN非常适合并行化计算，但是会丢失order信息，即使embed position信息还是不如RNN捕捉得好，而且CNN参数过多不好训练。RNN还有一个缺陷就是不能encode中间短语，只能从开头到结尾，导致最后一个单词在encode输出的hidden state权重过大，CNN可以编码任意N-gram的phrase，但是不是所有phrase都符合syntax，造成无效训练，这时候就提出Tree Recursive Neural Networks，和RNN很像，我的理解就是RNN展开是一个列表，那么TRNN展开就是树，都有递归属性。TRNN需要parse tree，因为这样才知道每一步该选取哪些phrase进行处理，我这次主要对Parsing with Compositional Vector Grammars这篇论文展开。<br/>

## Parsing with Compositional Vector Grammars
<br>这篇论文非常神奇地将parse task和phrase meaning task两个任务结合在一起了。parse task将sentence parse成continuous tree，phrase meaning task计算每个中间节点的represention，即不同phrase的meaning。论文提出naive TRNN，一般来说输入的数据包括sentence的word embedding还有word的POS tags，在naive TRNN里没有用到POS tag，在每一次运行时候，都计算可能的相邻两个phrase（论文给出的parse tree都是binary tree）的score，选取最大的组合合并成一个（好像huffman树啊，只是parse tree只能是相邻的），最开始就是word和word组合喽，越往上就是越大的phrase，注意此时学习的参数矩阵都是一样的，即使是不同词性的也是如此，就这样不停迭代直到只剩下一个节点为止。<br/>
<br>论文提出的loss也很有趣--max margin loss。对了，提一下网络里的实现细节，每个节点的简单将两个输入concatenate在一起，矩阵相乘，加上non-linearity即可，注意哦，两个输入没有任何交互，就得到这个节点的represention了，再将represention和向量参数相乘，就得到score了。loss的score是所有节点的score值相加。loss保证真实的parse tree的score最大，定义margin为predicted tree和real tree具有不相同label的节点（我也不知道这个label是啥囧），然后让所有predicted tree的score加上margin，取其中最大的，也就是最不像real tree的树，降低它的score值。通过最小化这个loss，来使正确的tree和错误的tree间隔足够大的margin。<br/>
<br>但是这个模型有个缺陷就是只使用一个简单的composition function来对所有词性组合的词语进行计算，换句话说，单个composition matrix在面对所有词性的词语还是太弱了点，当然可以采取多层neural network，但是就变得很难训练，所以论文针对不同的词性组合采取不同的composition matrix，在这里输入word的词性就有用了，但是我没理解之后的词性是怎么计算出来的，是PCEG直接给出的吗？但是score还加入了生成对应词性的概率，也就是说不是确定的，待填。论文提出beam search来选取每一步的组合。<br/>

## Supplement
<br>TRNN可以直接运用在具体任务里，比如判断每个phrase的sentiment，在输出meaning之后加上一层softmax就可以了。针对上节提出的输入值无法交互，这篇论文给出一种方法，模型输入的word不仅用一个vector还用一个matrix表示，在计算输出值时候，vector使得两个输入的vector和matrix相乘，matrix则是两个matrix concatenate。这样就可以解决输入值无法相互影响的问题了，但是很容易就可以发现这样训练的参数太大，每个word都有个matrix。论文提出另一个方法，就是只在vector和vector相乘加上另一个矩阵，生成和输入vector同样维度的输出。<br/>
<br>在LSTM流行之后，有学者就想能不能把tree structure结合原始LSTM结构，也可以把原始LSTM结构看作tree structure的特殊例子，tree和list区别就在于输入的个数。论文提出两种模型，Child-Sum Tree-LSTMs，简单将输入hidden state average，forget gate对每个输入都计算了一次，其他的和原始LSTM相同，非常好的适应dependency tree。另一种就是N-ary Tree-LSTMs，它限制输入的最大个数为N，而且输入是被ordered，计算每个gate时候，对待不同order的hidden state，有着不同的参数计算，然后把它们加在一起。特别指出计算单个输入的forget gate，它在该输入和不同chidren node之间都有参数，为了获取该输入和其它node的交互。这种模式很适合constituency tree，只要设置N=2即可。<br/>