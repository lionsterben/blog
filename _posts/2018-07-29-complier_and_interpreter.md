---
layout:     post
title:      "Compiler and Interpreter "
subtitle:   ""
date:       2018-08-20 22:26:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - programming language
---

<br>哎呀呀，这篇博文已经快翘了一个月了，是时候补上了。首先先表示对于这方面我只是个菜鸟，这篇文章不是具体将compiler和interpreter具体细节，主要是纠正自己以前错误的想法，大家想深入学习，需要啃些大部头了，看以后有没有时间或者需求，自己再往里面钻吧。<br/>
>A one-sentence sermon: Interpreter versus compiler is a feature of a particular programming-language implementation, not a feature of the programming language. One of the more annoying and widespread misconceptions in computer science is that there are "compiled languages" such as C and "interpreted languages" such as Racket. This is nonsense: I can write an interpreter for C or a compiler for Racket. (In fact, DrRacket takes a hybrid approach not unlike Java.) There is a long history of C being implemented with compilers and functional languages being implemented with interpreters, but compilers for functional languages have been around for decades. SML/NJ, for example, compiles each module/binding to binary code.

<br>首先需要明晰地是关于将编程语言分为compiler language和interpreter language是完全nonsense的做法，关于compile还是interprer是语言实现层面的问题，不是语言本身的问题。很多语言实现都是compiler和interprer混用。这里简单举python用Cpython实现的例子，如下图，python首先将自己编译成字节码，很像java的字节码，然后放进Cpython这个用C语言实现的虚拟机一条一条地解释执行，也就是说CPython实现方式同时利用了编译和解释方法。<br/>
```python
import dis

def love(girl):
    print("I love"+girl)

print(dis.dis(love))

result:
  5           0 LOAD_GLOBAL              0 (print)
              2 LOAD_CONST               1 ('I love')
              4 LOAD_FAST                0 (girl)
              6 BINARY_ADD
              8 CALL_FUNCTION            1
             10 POP_TOP
             12 LOAD_CONST               0 (None)
             14 RETURN_VALUE
None
```
>There are basically two approaches to this rest-of-the-implementation for implementing some programming language B. First, we could write an interpreter in another language A that takes programs in B and produces answers. Calling such a program in A an "evaluator for B" or an "executor for B" probably makes more sense, but "interpreter for B" has been standard terminology for decades. Second, we could write a compiler in another language A that takes programs in B and produces equivalent programs in some other language C (not the language C necessarily) and then uses some pre-existing implementation for C. For compilation, we call B the source language and C the target language. A better term than "compiler" would be "translator" but again the term compiler is ubiquitous. For either the interpreter approach or the compiler approach, we call A, the language in which we are writing the implementation of B, the metalanguage. 

<br>上述从PL课程讲义截取的文字清晰地表达compile和interpre的含义，compiler应该称为translater，如果说C语言将A语言翻译成B语言，那么C语言就称之为translator即翻译者，A语言是source language，B语言是target language，它只是起到翻译的作用，可以将Java和python翻译成字节码，如果愿意地话，你可以将python翻译成C语言。interpreter应该称为executor或者evaluator，执行器，我们可以用A语言写一个解释器来执行B语言的程序，一个自己写的小型解释器[here](https://github.com/lionsterben/Coursera/blob/master/uw_programming_languages/hw5.rkt)。<br/>
<br>在StackOverflow里看到一个很棒的解释，他用菜谱来表示程序执行，如果开始菜谱是由英文写作得，一个程序将英文翻译成中文，那个程序就是compiler，将这个中文的菜谱给机器人炒菜，机器人一个个指令的做下去，那么这个机器人就是interpreter。<br/>