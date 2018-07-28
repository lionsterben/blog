---
layout:     post
title:      "Closure Summary "
subtitle:   "closure summary"
date:       2018-07-28 22:24:00
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
我是从PL这门课学到的闭包概念，最开始是在SML这一函数式语言里，函数式语言里函数是第一等公民，代表着函数可以作为值被传递，被返回，被赋值，不知道为何肚子突然痛了，下次再写
