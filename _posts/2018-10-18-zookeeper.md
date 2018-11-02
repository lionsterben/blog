---
layout:     post
title:      "ZooKeeper: Wait-free coordination for Internet-scale systems"
subtitle:   "paper reading"
date:       2018-10-18 15:40:00
author:     "Dawei"
header-img: img/planet_earth_4k.jpg
tags:
    - paper reading
---

zookeeper leader make conf change, fobidden other clients use, set a ready znode, only when it exists. leader issue many update requests and final update ready node, if don't exist FIFO order, it may cause ready appear before other writes finish