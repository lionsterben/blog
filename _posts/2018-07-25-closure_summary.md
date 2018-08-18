---
layout:     post
title:      "Closure Summary "
subtitle:   "closure summary"
date:       2018-07-29 17:35:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - 技术随想
---
Closure Summary
==

## 何为closure  
closure中文名闭包，直接解释就是在支持闭包的语言中，函数并不只是那些code，是包括code和environment的结合体，闭包需要词法作用域的支持，也就是函数内部引用的自由变量是在函数定义的变量，而不是运行时的变量。这样闭包就由函数代码和函数定义时的环境组成。

## environment
可以看到闭包的精髓就在于环境如何保存，具体可以参见R大这个回答[here](https://www.zhihu.com/question/27416568/answer/36565794),总之有两种，capture by value和capture by reference,值捕获就是将需要的自由变量复制一份到对象内部，java是这样支持的，所以它所用的自由变量只支持final类型的变量，防止变量已经改变，闭包做不出反应。另一种是引用捕获，闭包内部拷贝的是变量的引用，也就是说闭包可以直接对自由变量进行改变，python实际上是引用闭包，但是不支持修改这些自由变量。

## 我的学习
我是从PL这门课学到的闭包概念，这门课程是在Coursera上华盛顿大学开的一门课程，链接在这[here](https://www.coursera.org/learn/programming-languages/)。最开始是在SML这一函数式语言里，函数式语言里函数是第一等公民，代表着函数可以作为值被传递，被返回，被赋值~~不知道为何肚子突然痛了，下次再写~~。还有一点，SML的数据是immutable的，所以在闭包环境上，不管是用引用捕获还是值捕获对于用户来说都一样。因为pure函数式语言里面只以函数表达计算，不像java这种OOP语言，有对象来保存状态，但是因为闭包的存在，函数也可以保存状态，所以两种类型的语言在表现力上是相同的。关于闭包，这门课程有个小作业，用racket实现一个小型解释器，其中就包括闭包功能，链接在这[here](https://github.com/lionsterben/Coursera/blob/master/uw_programming_languages/hw5.rkt)，有兴趣的小伙伴可以看一看，总之就是在解释器eval语句时，遇到function，就把它包装成一个包含当前环境的Closure返回，在Call的时候直接eval这个closure。关于解释器和编译器还可以写一篇博文。总之先这样吧。对了，有个疑惑，如果一个语言数据可变，实现闭包的时候是用引用捕获，那加入有多个函数都捕获到这个变量，那一个函数改变，那岂不是其他所有闭包内的这个自由变量值全变了，很不可控啊，还需要学习一个。
