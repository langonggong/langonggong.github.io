---
title: Damerau–Levenshtein distance
description: Damerau–Levenshtein distance
date: 2019-11-16 18:06:22
keywords: Elasticsearch,Lucune
categories : [Elasticsearch]
tags : [Elasticsearch, 编辑距离, Lucune]
comments: true
---

# 算法介绍

Damerau–Levenshtein distance（D氏距离）对比L氏距离，还新增了交换操作（响铃两个字符的交换），一共包含了新增、删除、替换、交换四种编辑操作。计算公式如下

<img src="/images/Damerau–Levenshtein distance公式.png">

该公式的最后一步，表示a字符串前i个字符序列可以通过相邻两个

# 示例

将abc转换为acb
Levenshtein Distance算法解决方法如下：
方法一：

<img src="/images/Damerau–Levenshtein distance示例步骤1.png">

abc->acc（将字符b替换成字符c）
acc->acb（将字符c替换成字符b）
一共进行两次字符替换操作
方法二：

<img src="/images/Damerau–Levenshtein distance示例步骤2.png">

abc->ac（删除字符b）
ac->acb（新增字符b）
先删除中间的字符b，再在末尾添加字符b
方法三：

<img src="/images/Damerau–Levenshtein distance示例步骤3.png">

abc->acbc（新增字符c）
acbc->acb（删除字符c）
先新增字符c，再删除末尾的字符c

Damerau–Levenshtein distance算法解决过程如下：

<img src="/images/Damerau–Levenshtein distance示例结果图.png">

将b字符和c字符替换即可，满足条件

<img src="/images/Damerau–Levenshtein distance交换.png">