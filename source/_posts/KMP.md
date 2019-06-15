---
title: KMP
description: KMP算法
date: 2018-08-30 00:44:12
keywords: 算法
categories : [算法, 字符串匹配]
tags : [算法, 字符串匹配]
comments: true
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 求next数组

字符串组成为
$$ p_0...p_k...p_i...p_{i+k}...p_j...p_{j+k}...p_m...p_{m+k}... $$
假设\\(next_{[i+k]}==next_{[m+k]}\\)，表示\\(p_0...p_{i+k}=p_j...p_{m+k}\\) 
- 如果\\(p_{i+k+1}=p_{m+k+1}\\)，则\\(next_{[m+k+1]}==next_{[m+k]}+1\\) 
- 如果\\(p_{i+k+1}!=p_{m+k+1}\\)，有\\(next_{[i+k]}=k\\)，表示\\(p_0...p_{k}=p_i...p_{i+k}=p_m...p_{m+k}\\)，
 - 如果\\(p_{m+k+1}=p_{k+1}\\)，则\\(next_{[m+k+1]}=next_{[k]}+1\\)
 - 如果\\(p_{m+k+1}!=p_{k+1}\\)，则有\\(next_{[k]}=l\\)，继续上面的过程

总结下来，若\\(next_{[i]}==next_{[j]}\\)
- if \\(p_{i+1}=p_{j+1}\\), then \\(next_{[j+1]}==next_{[j]+1}\\)
- else \\(i==next_{[i]}\\),继续上一步

# 实现代码
```
public class KMP {

  private int[] getNextArr(char[] pattern) {
    if (pattern == null || pattern.length == 1) {
      return new int[]{-1};
    }
    int[] next = new int[pattern.length];
    next[0] = -1;
    next[1] = 0;
    int currentNext=0;
    int pos = 2;
    while (pos < pattern.length) {
      if (pattern[pos - 1] == pattern[currentNext]) {
        next[pos++] = ++currentNext;
      } else if (currentNext > 0) {
        currentNext = next[currentNext];
      } else {
        next[pos++] = 0;
      }
    }
    return next;
  }

  public int getIndexOf(String ori, String pattern) {
    if (ori.length() < pattern.length()) {
      return -1;
    }
    char[] oriArr = ori.toCharArray();
    char[] patternArr = pattern.toCharArray();

    int indexOfOri = 0;
    int indexOfPattern = 0;

    int[] next = getNextArr(patternArr);

    while (indexOfOri < oriArr.length && indexOfPattern < patternArr.length) {
      if (oriArr[indexOfOri] == patternArr[indexOfPattern]) {
        indexOfOri++;
        indexOfPattern++;
      } else if (next[indexOfPattern] >= 0) {
        indexOfPattern = next[indexOfPattern];
      } else {
        indexOfOri++;
      }
    }
    return indexOfPattern == patternArr.length ? indexOfOri - indexOfPattern : -1;
  }

}
```