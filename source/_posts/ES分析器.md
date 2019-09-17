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

### Edge NGram Token Filter
n-gram 看成一个在词语上 滑动窗口 ， n 代表这个 “窗口” 的长度。如果我们要 n-gram quick 这个词 —— 它的结果取决于 n 的选择长度：

```
长度 1（unigram）： 	[ q, u, i, c, k ]
长度 2（bigram）： 	[ qu, ui, ic, ck ]
长度 3（trigram）： 	[ qui, uic, ick ]
长度 4（four-gram）：	[ quic, uick ]
长度 5（five-gram）：	[ quick ]
```
朴素的n-gram对词语内部的匹配非常有用，即在 [Ngram匹配复合词](https://www.elastic.co/guide/cn/elasticsearch/guide/current/ngrams-compound-words.html) 介绍的那样。但对于输入即搜索（search-as-you-type）这种应用场景，我们会使用一种特殊的n-gram称为边界n-grams（edge n-grams）。所谓的边界n-gram是说它会固定词语开始的一边，以单词quick为例，它的边界n-gram的结果为：

```
[q,qu,qui,quic,quick]
```
创建索引、实例化 token 过滤器和分析器的完整示例如下

```
PUT /my_index
{
    "settings": {
        "number_of_shards": 1, 
        "analysis": {
            "filter": {
                "autocomplete_filter": { 
                    "type":     "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "autocomplete": {
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "autocomplete_filter" 
                    ]
                }
            }
        }
    }
}
```
可以拿 analyze API 测试这个新的分析器确保它行为正确：

```
GET /my_index/_analyze
{
  "text": "quick brown",
  "analyzer": "autocomplete"
}
```
结果表明分析器能正确工作，并返回以下词：

```
[q,qu,qui,quic,quick,b,br,bro,brow,brown]
```
可以用 update-mapping API 将这个分析器应用到具体字段并添加数据

```
PUT /my_index/_mapping/
{
  "properties": {
    "name": {
      "type": "text",
      "analyzer": "autocomplete"
    }
  }
}

POST /my_index/_bulk
{"index":{"_id":1}}
{"name":"Brown foxes"}
{"index":{"_id":2}}
{"name":"Yellow furballs"}
```
如果使用简单 match 查询测试查询 “brown fo” ,可以看到两个文档同时都能匹配，尽管 Yellow furballs 这个文档并不包含 brown 和 fo

```
GET /my_index/_validate/query?explain
{
    "query": {
        "match": {
            "name": "brown fo"
        }
    }
}
```
使用validate-query API 分析：

```
GET /my_index/_validate/query?explain
{
  "query": {
    "match": {
      "name": "brown fo"
    }
  }
}
```
explanation 表明查询会查找边界 n-grams 里的每个词：

```
name:b name:br name:bro name:brow name:brown name:f name:fo
```
name:f 条件可以满足第二个文档

我们需要保证倒排索引表中包含边界 n-grams 的每个词，但是我们只想匹配用户输入的完整词组（ brown 和 fo ）， 可以通过在索引时使用 autocomplete 分析器，并在搜索时使用 standard 标准分析器来实现这种想法，只要改变查询使用的搜索分析器 analyzer 参数即可：

```
GET /my_index/_search
{
  "query": {
    "match": {
      "name": {
        "query": "brown fo",
        "analyzer": "standard"
      }
    }
  }
}
```
换种方式，我们可以在映射中，为name字段分别指定index_analyzer和 earch_analyzer。因为我们只想改变 search_analyzer，这里只要更新现有的映射而不用对数据重新创建索引：

```
PUT /my_index/_mapping
{
  "properties": {
    "name": {
      "type": "text",
      "analyzer": "autocomplete",
      "search_analyzer": "standard"
    }
  }
}
```
如果再次请求 validate-query API ，当前的解释为：

```
name:brown name:fo
```
再次执行查询就能正确返回 Brown foxes 这个文档。
## 分析