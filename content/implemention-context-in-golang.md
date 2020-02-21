---
title: "[GO]Context的golang实现"
date: 2020-02-21T15:32:54+08:00
draft: true
---

context是golang里面的标准库，在go里面到处都可以找到使用ctx的例子，网上有很多context的科普文章
其中也有比较写的比较好的，这部分我会贴在文章末尾。今天我们主要是带着大家如何从0到1实现一个ctx，有了这种实践之后
具体在go源码中的使用场景分析，我会放在下一篇blog中，并且结合[golang数据库连接池实现](http://www.youmakemeday.com/parse-implementation-of-golang-database-sql-connection/)一起探究其中的奥秘，确保能一举深刻的理解ctx。
<!--more-->

