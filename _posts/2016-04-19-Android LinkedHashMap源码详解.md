---
layout: post
title: "Android LinkedHashMap源码详解"
subtitle: 'Android LinkedHashMap源码详解'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “灯前目力虽非昔，犹课蝇头二万言。”
	--陆游

## 正文

在上一篇中我们分析了HashMap的源码，了解HashMap是以数组加链表的形式存储的，这一篇我们结合上一篇的内容来分析一下LinkedHashMap的源码，在阅读之前最好能把上一篇的<a href="https://androidboke.com/2016/04/15/Android-HashMap%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3" target="_blank">Android HashMap源码详解</a>看一遍，尤其是HashMap的结构图要理解清楚，我们来先看一下LinkedHashMap的构造方法，由于比较多，我们随便挑一个