---
layout:     post
title:      "UNSUPERVISED MACHINE TRANSLATION USING MONOLINGUAL CORPORA ONLY"
subtitle:   "paper reading"
date:       2018-03-06 11:45:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---

<br>这篇论文介绍FAIR利用非对齐两种语言数据训练机器翻译模型，不管从模型效果还是模型设计都很有趣，论文指出该模型在英文到法文的翻译能够达到32.76的BLEU值，当然效果肯定比不上拥有平行语料的模型，但是对于那些没有或者很难收集大规模平行语料的语言翻译，论文介绍的方法还是很有意义的，毕竟收集两种语言的语料和收集平行语料的难度和工作量不是一个量级的。<br/>

<br>模型架构：和普通的翻译系统一样，模型也分为encoder和decoder，encoder将源语言或者目标语言映射到一个隐空间，decoder则是从隐空间映射回目标语言或者源语言，这里注意模型只有一个encoder和一个decoder。encoder使用对应语言的embedding计算每个单词对应的hidden state，decoder根据上一步hidden state，上一步预测单词和encoder states加权平均的context vector得出当前单词，问题：decoder开始时候初始化hidden state是encoder的最后输出吗？。论文指出encoder使用3层双向LSTM，decoder使用3层单向LSTM，感觉就是原始seq2seq+attention的模型。。。<br/>

<br>ok，先介绍一下方法，我们用src表示源语言数据集，tar表示目标语言数据集，当然它们不是对齐的，我们会有三个任务,这三个任务都是对称的，src和tar都可以作为输入，第一个给定某一语言的噪声输入，经过encoder映射到隐空间，然后decoder从隐空间返回该语言域；第二个，假设给定是src，根据上一步的模型翻译成tar，然后加噪声，encoder到隐空间，decoder到src，比较和初始值距离，如果给定是tar就是对称的；第三个，为了使src和tar域在隐空间对齐，我的理解：因为encoder在src和tar域里是共享的，所以希望即使是不同语言也可以映射到同一个空间？（还是不太了解，mark一下），做法和GAN很像，训练一个判别器：根据隐空间的状态判断是来自src还是tar，判断器尽可能将它们区分出来。encoder则尽可能愚弄（我不知道怎么翻译了。。）判别器，在给定判别器参数后，我们希望将src的encoder结果被判定为tar，反之亦然。这样不同迭代后，encoder会被训练成能够将src和tar的映射结果空间相同。<br/>

<br>第一个任务是要模型学习对自己的自编码，通过将自身映射到隐空间，在从隐空间返回原域，使得模型学会语言域到隐空间的双向转换，但是这没有学会两个语言域的转换。第二个任务先将源语言翻译成目标语言，再加噪声，转换成源语言，让模型学会从一个语言域到另一个语言域的变换。论文指出如果encoder将两种语言的隐空间表示都表示为一种很强的结构，那么也会加强模型效果(不太理解啊)，是这样吗：假设输入是src，第一个任务是将src encode到隐空间里，再decode回src域，第二个任务是将src翻译的tar encode到隐空间，之后再decode到src域，可以看到encoder输入域是不同的，但是decode的输入都是相同的，我们希望即使输入域不同，encoder也可以将它们映射到隐空间相同位置，也就是说encoder对不同语言的输入尽可能产生相同的结构。<br/>

<br>最后介绍一下细节和训练流程，加噪声的操作一个是随机丢弃单词，第二个打乱原始语序，具体细节参见论文2.3节。模型最后的损失函数是上述三个任务loss的加权平均，权重是自己定的超参数。训练开始后用字典单词一一对应的翻译初始化模型，之后每一步利用当前模型翻译两种语言数据集，最小化判别器loss，最小化三任务loss，将当前encoder和decoder作为下一步翻译模型，直至收敛。还有个问题，没有显示训练效果的标准，论文使用src数据翻译成tar，tar再翻译回src的BLEU值，tar数据亦然。<br/>