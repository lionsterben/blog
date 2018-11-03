---
layout:     post
title:      "ELMo and BERT"
subtitle:   "paper reading"
date:       2018-10-25 13:31:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---   

<br>在NLP界，迁移学习不像cv那么普遍，针对每个任务，不同学者会提出不同的模型进行处理，最多的是在word embedding层使用相同的word vector，但是这些embedding不能进行词语的消歧义，即在不同语境下同一个词语应该表示成不同的意思，ELMO就是通过在大规模无监督预料上训练提取句子的词embedding，再喂进下游任务来解决这个问题。但是这种方式和之前的pretraining word vector相同，只是在input层增强表示能力，有人指出其实各种各样的NLP任务核心都应该是对语言本身建模，就想cv界对图像进行抽取特征，于是BERT模型诞生，训练出能够表示语言word，syntax，sentence各种层面意义的模型，对于各种nlp task，在最后加一层softmax layer就可以达到SOTA的效果，令人叹为观止。所以这篇博客就主要介绍Deep contextualized word representations和BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding这两篇论文。<br/>

# Deep contextualized word representations
<br>论文提出新的Deep contextualized word representations，用来解决两个问题：1.word的复杂特征表示（比如句法和语义上的）2.在不同上下文词语表示问题。论文通过对在大规模语料（非常容易获取）训练，得到能够出色对句子进行表示的模型，注意哦，这个模型得到的是模型本身，而不是word embedding。<br/>
<br>模型本身非常简单，多层双向LSTM language model(biLM)，每个word由句子里前面的word和后面的word分别预测，将每层的hidden state concatenate在一起表示这个这层句子的此word的表示。论文指出训练出来的模型所输出的state在不同层可以表示不同的意义，在低层可以对句法进行建模，高层可以捕捉上下文的word meaning，这就成功解决提出的问题了。<br/>
<br>论文给出了如何将ELMO模型应用在下游任务，其实本质来说就是在下游任务的input层面进行改进，首先固定住所有biLM模型的参数，不让它们参与训练，然后让sentence过一遍biLM模型，得到不同layer的state表示，这时候每个word的表示就是不同层的加权平均，而且针对不同的任务需要乘以一个权重（因为biLM模型内部状态和task specific的表示有不同），这样就得到ELMO表示了，在将得到的word represention和初始的word embeddeding concatenate在一起喂进task specific模型里就可以了，通过训练调节biLM不同layer的权重来针对不同的任务获取word不同层面的信息。<br/>
<br>这个模型也有一定的缺陷，lstm本身训练较慢加上如此大规模的语料，造成训练计算成本非常大，所以论文也只给出两层layer的模型，同时对内部状态也进行删减。不过模型效果还是很不错的。接下来看下Google的人在掌握足量算力资源后干出来丧心病狂的事情。<br/>

# BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding
<br>ELMO通过双向biLM来对语言进行建模，但是并不是真正的双向，也就说预测word前面的word和后面的word并没有产生真正的交互，而多层lstm也不支持这样的交互（在上层会直接泄露预测word的信息）。所以BERT采取Attention is all you need里的transformer attention机制，对需要预测的词直接覆盖掉，不显示（实际操作word替换为mask表示），从而达到双向self attention效果。这样做的好处一是transformer采取直接的矩阵相乘，比lstm速度不知道高到哪里去了，二是真正的双向建模，每个word都和所有word进行交互。<br/>
<br>模型整体就是多个transformer叠加在一起，主要说以下论文提出在这个模型上运行的任务，第一个就是之前提到的预测mask word任务（Masked LM），论文抽取15%的单词进行mask处理，但是这样做的话会导致pretrain和fine tune的时候造成偏差（毕竟fine tune可没有mask），所以对mask word进行调整，80%mask，10%用随机单词替换，10%不变（让表示偏向实际观测到的值，没太懂，抵消随机单词替换的影响？），训练的时候不知道哪个单词被替换了，所以对所有input token都进行预测。第二个问题就是每次只有15%的单词被预测，也就是训练，导致需要更多的步骤来收敛。第二个任务是在sentence层面的，因为很多任务比如QA和NLI需要理解两个句子之间的关系，但是这个不能在language modeling层面捕捉，所以论文提出next sentence prediction任务，给出两个句子，预测它们实际是不是连续的。<br/>
<br>这里提一下输入的格式问题，输入可以是单个句子或者一对句子，每个input开头插入一个cls标记，用于句子的分类任务（可以把它输出的state看成聚合整个句子的信息），一对句子用sep标记分离，所以每个句子input token还增加了cls和sep（sep仅限于sentence pair），这样每个input token就由三部分构成：token embeddeding，segment embeddeding（表示属于不同句子），position embeddeding（表示在句子中的位置）。<br/>
<br>这样对每个mask的预测和cls标志判定next sentence label训练就好了。具体的超参数设置我就不说了，感兴趣的小伙伴可以看论文，只是想感慨一句，64块TPU训练4天，有钱真好。最神奇的就是这个模型可以直接作为不同nlp task的主干模型，就和图像里用resnet作为主干一样，一般在最后对word token或者cls token进行一层softmax layer就可以了，然后论文作者靠这个模型刷新了11个？nlp task的SOTA，太厉害了。<br/>
<br>也有人质疑也许模型表现得那么好，不是因为model本身很棒，只是训练集变大，能够训练出更好的效果而已，或者其它的model我拿那么多的资源和数据集去训练也可以，还是要看之后的进展，Google已经把BERT开源了，我在跟进跟进吧，复现出论文效果是不能了，毕竟有人算过了，训练一次在Google cloud上至少750美刀。。。<br/>