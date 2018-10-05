---
layout:     post
title:      "Race condition and data race"
subtitle:   "project thinking"
date:       2018-09-28 09:40:00
author:     "Dawei"
header-img: img/starry_night_sky.jpg
tags:
    - project thinking
---
<br>ok，在又看了几篇博客后感觉可以写一下了，像这种博文一般是自己老是拎不清，但是重复遇到的问题，以后也会多写一些这些文章。<br/>
<br>这篇博客的主要思路来自于这篇[blog](https://blog.regehr.org/archives/490),我也加入一些自己的理解进去。首先介绍下race condition和data race的定义。
>race condition is a flaw that occurs when the timing or ordering of events affects a program’s correctness. Generally speaking, some kind of external timing or ordering non-determinism is needed to produce a race condition; typical examples are context switches, OS signals, memory operations on a multiprocessor, and hardware interrupts.  
Data race: A data race is a situation, in which at least two threads access a shared variable at the same time. At least on thread tries to modify the variable.    

race condition侧重于因为执行指令的顺序和时间不同对结果的正确性影响，data race则是在执行过程中对同一变量并发访问，而且至少有一个改变该变量，并发是指没有类似锁的机制，使得某一线程happen before或者after另一个线程的操作。实际上，的确这两者有很大的重叠，data race导致race condition的情况很多，但是也有两者互相独立的情况。在这里我的理解是data race更侧重单个operation的结果，race condition则是整个程序的结果是否正确。<br/>
<br>这一节举几个例子，例子也是来源于这篇[blog](https://blog.regehr.org/archives/490)
```python
transfer1 (amount, account_from, account_to) {
  if (account_from.balance < amount) return NOPE;
  account_to.balance += amount;
  account_from.balance -= amount;
  return YEP;
}
```
这是典型的data race和race condition都存在的程序，因为变量没有锁住，两个线程可以同时读取变量值，然后对其修改，再写回去，导致最后结果错误。解决data race很简单，把语句的变量原子化操作就可以了。
```python
transfer2 (amount, account_from, account_to) {
  atomic {
    bal = account_from.balance;
  }
  if (bal < amount) return NOPE;
  atomic {
    account_to.balance += amount;
  }
  atomic {
    account_from.balance -= amount;
  }
  return YEP;
}
```
这样处理的话，每次访问变量就有明晰的happen before关系，不存在同时访问变量的情况了，但是race condition还是存在，也就是在某一时刻，另一个线程可以发现金钱总数发生变化，比如两个线程执行互相转账，加入都进行到account_to.balance += amount之后，另一个线程就会发现总钱数多了，发生错误。
```python
transfer3 (amount, account_from, account_to) {
  atomic {
    if (account_from.balance < amount) return NOPE;
    account_to.balance += amount;
    account_from.balance -= amount;
    return YEP;
  }
}
```
为了解决race condition造成的结果不对，将全部语句原子化操作（我觉得很像解决死锁时候，将所有冲突变量的锁拿到手），每个线程只有执行完这段代码，才会释放变量的锁，这样中间状态就不会不同了。当然也存在不是race condition，但是有data race的情况出现。
```python
transfer4 (amount, account_from, account_to) {
  account_from.activity = true;
  account_to.activity = true;
  atomic {
    if (account_from.balance < amount) return NOPE;
    account_to.balance += amount;
    account_from.balance -= amount;
    return YEP;
  }
}
```
我们在代码里设置两个标记，会出现data race，但是对程序运行可能是完全无害而且也不存在race condition。

  