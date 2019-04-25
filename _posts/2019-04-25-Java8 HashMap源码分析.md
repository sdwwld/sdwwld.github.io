---
layout: post
title: "Java8 HashMap源码分析"
subtitle: '深入分析Java8中的HashMap源码'
author: "山大王"
header-style: text
tags:
  - 源码
  - android
---

之前分析过java7的HashMap的原理，其实java7和java8对HashMap的改动还是比较大的，java7中HashMap使用的是数组和链表的形式存储的，这里可以看一下之前写的[HashMap源码详解](https://blog.csdn.net/abcdef314159/article/details/51165630)，而java8对HashMap的改动是当链表的长度大于等于8的时候就会把它转化为红黑树，之前也特意分析过红黑树的源码，[TreeMap红黑树源码详解](https://blog.csdn.net/abcdef314159/article/details/77193888)，但java8中使用红黑树不是TreeMap，是TreeNode，他的原理和TreeMap类似，待会我们看下代码。