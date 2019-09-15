---
title: ES分析器
description: ES分析器，由字符过滤器、分词器、分词过滤器组成
date: 2019-09-15 01:09:03
keywords: Elasticsearch
categories : [Elasticsearch]
tags : [Elasticsearch, 分析器]
comments: true
---

# 分析器
# CharFilter
# Tokenizer
# Tokenizer Filter
## 转换

### Lowercase Token Filter
小写过滤器。将标记token规范化为小写
### Uppercase Token Filter
大写过滤器。将术语转写成大写

### ASCII Folding Token Filter

ASCII码折叠过滤器。asciifolding过滤器将ASCII码不在ASCII表前127内的字母、数字和Unicode符号转换为ASCII等效字符(如果存在的话)

``` 
//新建索引
PUT asciifold_example
{
    "settings": {
        "analysis": {
            "analyzer": {
                "my_analyzer": {
                    "tokenizer": "standard",
                    "filter": [
                        "my_ascii_folding"
                    ]
                }
            },
            "filter": {
                "my_ascii_folding": {
                    "type": "asciifolding",
                    "preserve_original": false
                }
            }
        }
    }
}
```

```
//提交文本
POST asciifold_example/_analyze
{
    "analyzer": "my_analyzer",
    "text": "Êê Ẑẑ Ĉĉ Ŝŝ Ŋŋ Öö"
}
```

```
//获取结果
[Ee,Zz,Cc,Ss,Nn,Oo]

```
## 过滤

### Length Token Filter
长度过滤器。会移除token流中太长或太短的标记

## 切割
## 分析