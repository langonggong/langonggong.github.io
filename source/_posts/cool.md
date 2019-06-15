---
title: hexo高级特性
description: 测试hexo系统新特性
date: 2018-08-19 16:35:51
keywords: hexo
categories : [hexo]
tags : [hexo, 新特性]
comments: true
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 网易云

## 通过外链添加
在网易云音乐的网页版生成歌单外链

```
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=450 src="//music.163.com/outchain/player?type=0&id=2343741251&auto=1&height=430"></iframe>
```

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=450 src="//music.163.com/outchain/player?type=0&id=2343741251&auto=1&height=430"></iframe>

## 通过aplayer插件
[MeingJS 支持](https://github.com/MoePlayer/hexo-tag-aplayer/blob/master/docs/README-zh_cn.md#meingjs-%E6%94%AF%E6%8C%81-30-%E6%96%B0%E5%8A%9F%E8%83%BD)
id为网页上url后面的id值
```
{% meting "468513829" "netease" "song" "autoplay" "mutex: true" "listmaxheight:340px" "preload:none" "theme:#ad7a86"%}
```
{% meting "468513829" "netease" "song" "autoplay" "mutex:true" "listmaxheight:340px" "preload:none" "theme:#ad7a86"%}
```
{% meting "384485381" "netease" "playlist" "autoplay" "mutex: true" "listmaxheight:340px" "preload:none" "theme:#ad7a86"%}
```
{% meting "2384642500" "netease" "playlist" "autoplay" "mutex: true" "listmaxheight:340px" "preload:none" "theme:#ad7a86"%}

## 行间公式

$$ 
f(n)=\begin{cases}
n/2, & \text{如果$ x<=2 $}\\
3n+1, & \text{如果$ x>2 $}
\end{cases}
$$

## 行内公式

比如 \\(x=\frac{-b\pm\sqrt{b^2-4ac}}{2a}\\)