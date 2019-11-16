---
title: Double Array Trie
description: Double Array Trie
date: 2019-11-16 17:20:56
keywords: Elasticsearch,Lucune
categories : [Elasticsearch]
tags : [Elasticsearch, 树, Lucune]
comments: true
---

# 问题
Trie树的创建要考虑的是父节点如何保存孩子节点，主要有链表和数组两种方式：

- 使用节点数组，因为是英文字符，可以用Node[26]来保存孩子节点(如果是数字我们可以用Node[10])，这种方式最快，但是并不是所有节点都会有很多孩子，所以这种方式浪费的空间太多
- 用一个链表根据需要动态添加节点。这样我们就可以省下不小的空间，但是缺点是搜索的时候需要遍历这个链表，增加了时间复杂度。
- Trie也可以按照DFA的方式存储，即表示为转移矩阵。行表示状态，列表示输入字符，(行, 列)位置表示转移状态。这种方式的查询效率很高，但由于稀疏的现象严重，空间利用效率很低。

如果是中文词典，子节点的数目范围跨度更大，问题将会更加严重

<img src="/images/字典树父子节点.png">

# 原理
双数组Trie树（Double-array Trie, DAT）是一种Trie树的高效实现，兼顾了查询效率与空间存储

Double-Array Trie包含base和check两个数组。base数组的每个元素表示一个Trie节点，即一个状态；check数组表示某个状态的前驱状态。
其中，s是当前状态的下标，t是转移状态的下标，c是输入字符的数值。如图所示：
base和check的关系满足下述条件：
base[s] + c = t
check[t] = s

<img src="/images/双数组.png">

以bachelor, baby, badge, jar为例

<img src="/images/双数组示例.png">

其中，字符的编码表为{'#'=1, 'a'=2, 'b'=3, 'c'=4, etc. }。为了对Trie做进一步的压缩，用tail数组存储无公共前缀的尾字符串，且满足如下的特点：
p = -base[m], tail[p] = b1, tail[p+1] = b2, ..., tail[p+h-1] = bh；
h为该尾字符串的长度

# 检索示例
检索词badge的过程如下：

```
// root -> b
base[1] + 'b' = 4 + 3 = 7
check[7]=1
// root -> b -> a
base[7] + 'a' = 1 + 2 = 3
check[3]=7
// root -> b -> a -> d
base[3] + 'd' = 1 + 5 = 6
check[6]=3
// badge#
base[6] = -12
tail[12..14] = 'ge#'
```

