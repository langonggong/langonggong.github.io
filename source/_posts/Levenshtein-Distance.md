---
title: Levenshtein Distance
description: Levenshtein Distance
date: 2019-11-16 17:46:00
keywords: Elasticsearch,Lucune
categories : [Elasticsearch]
tags : [Elasticsearch, 编辑距离, Lucune]
comments: true
---

# 算法介绍
Levenshtein Distance（L氏距离）是一个度量两个字符序列之间差异的字符串度量标准，两个单词之间的Levenshtein Distance是将一个单词转换为另一个单词所需的单字符编辑（插入、删除或替换）的最小数量

例如：将单词abroad转换为aboarc

- abroad->aboad（删除字符r）
- aboad->aboard（插入字符r）
- aboard->aboarc（将字符d替换为字符c）

编辑距离为3

# 算法实现

对于两个字符串a、b，长度分别为|a|、|b|，i、j分别为a、b的下标值，它们的Levenshtein Distance 为：

<img src="/images/Levenshtein Distance公式.png">

表示字符串a的前i个字符序列转换为字符串b的前j个字符序列所需要的最短的编辑距离

1、空字符串与任意非空字符串的编剧距离等于非空字符串的长度

<img src="/images/Levenshtein Distance空字符.png">

<img src="/images/Levenshtein Distance空字符矩阵.png">

如图第二行，将空字符串转换为k需要插入一个字符，编辑距离为1；
将空字符串转化为kf，需要插入两个字符，编辑距离为2；
如图第二列，将k转换为空字符串，需要删除一个字符，编辑距离为1；
将kc转换为空字符串，需要删除两个字符，编辑距离为2

2、删除一个字符

<img src="/images/Levenshtein Distance删除.png">

<img src="/images/Levenshtein Distance删除矩阵.png">

```
a="kfc",b="kc",i=2,j=1,
lev(1,1)等同于将字符串a中的"k"转换为字符串b中的"k"，编辑距离为0
lev(2,1)等同于将字符串a中的"kf"转换为字符串b中的"k"，编辑距离为1，需要一步删除操作
lev(i,j)=lev(2,1)=lev(1,1)+1=0+1=1
```

由图像可知：在前一行的编辑距离上加一

3、新增一个字符

<img src="/images/Levenshtein Distance新增.png">

<img src="/images/Levenshtein Distance新增矩阵.png">

```
a="kc",b="kfc",i=1,j=2
lev(1,1)等同于将字符串a中的"k"转换为字符串b中的"k"，编辑距离为0
lev(1,2)等同于将字符串a中的"k"转换为字符串b中的"kf"，编辑距离为1，需要一步新增操作
lev(i,j)=lev(1,2)=lev(1,1)+1=1
```

由图像可知：在前一列的编辑距离上加一

4、当前两个字符相同，编辑距离不变

<img src="/images/Levenshtein Distance不变.png">

<img src="/images/Levenshtein Distance不变矩阵.png">

```
a="kc",b="kfc",i=2,j=3
lev(1,2)等同于将字符串a中的"k"转换为字符串b中的"kf"，编辑距离为1，需要一步新增操作
lev(2,3)等同于将字符串a中的"kc"转换为字符串b中的"kfc"，编辑距离为1
因为字符串a中的c字符等于字符串b中的c字符，所以lev(i,j)=lev(2,3)=lev(1,2)+0
```

由图像可知：与左上角坐标所示编辑距离相同

5、当前两个字符不同，替换操作

<img src="/images/Levenshtein Distance替换.png">

<img src="/images/Levenshtein Distance替换矩阵.png">

```
a="kc",b="kb",i=j=2
lev(1,1)等同于将字符串a中的"k"转换为字符串b中的"k"，编辑距离为0
lev(2,2)等同于将字符串a中的"kc"转换为字符串b中的"kb"，编辑距离为1，需要替换一个字符串
```

由图像可知：将左上角坐标所示编辑距离加一

# 示例

求最优编辑距离示例：将单词abroad转换为aboarc
1、将abr转换为ab，删除字符r

<img src="/images/Levenshtein Distance示例步骤1.png">

2、将aboa（之前已删除字符r）转换为aboar，新增字符r

<img src="/images/Levenshtein Distance示例步骤2.png">

3、将aboard转换为aboarc，将字符d替换为字符c

<img src="/images/Levenshtein Distance示例步骤3.png">

# 字符串的相似度

<img src="/images/Levenshtein Distance字符串相似度.png">

# 应用

DNA序列分析、拼写检查、语音辨识、抄袭侦测


