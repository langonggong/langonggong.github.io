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

### 词干提取器

[词干提取器](http://langonggong.com/2019/09/21/%E8%AF%8D%E5%B9%B2%E6%8F%90%E5%8F%96%E5%99%A8/)将一个词提取为它的词根形式
## 过滤

### Length Token Filter
长度过滤器。会移除token流中太长或太短的标记

## 切割

### ngram

[链接](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_ngrams_for_partial_matching.html)
朴素的n-gram对词语内部的匹配非常有用，但对于输入即搜索（search-as-you-type）这种应用场景，我们会使用一种特殊的n-gram称为边界n-grams（edge n-grams）


###滑动窗口
使用 TF/IDF 的标准全文检索将文档或者文档中的字段作一大袋的词语处理。 match 查询可以告知我们这大袋子中是否包含查询的词条，但却无法告知词语之间的关系。

思考下面这几个句子的不同：

- Sue ate the alligator.
- The alligator ate Sue.
- Sue never goes anywhere without her alligator-skin purse.

用match搜索sue alligator上面的三个文档都会得到匹配，但它却不能确定这两个词是否只来自于一种语境，甚至都不能确定是否来自于同一个段落。

理解分词之间的关系是一个复杂的难题，我们也无法通过换一种查询方式去解决。但我们至少可以通过出现在彼此附近或者仅仅是彼此相邻的分词来判断一些似乎相关的分词。

对句子Sue ate the alligator ，不仅要将每一个单词（或者 unigram ）作为词项索引
```
["sue", "ate", "the", "alligator"]
```
也要将每个单词以及它的邻近词作为单个词项索引：
```
["sue ate", "ate the", "the alligator"]
```
这些单词对（或者 bigrams ）被称为 shingles 。
你也可以索引三个单词（ trigrams ）。Trigrams 提供了更高的精度，但是也大大增加了索引中唯一词项的数量。在大多数情况下，Bigrams 就够了。
```
["sue ate the", "ate the alligator"]
```
只有当用户输入的查询内容和在原始文档中顺序相同时，shingles 才是有用的；对 sue alligator 的查询可能会匹配到单个单词，但是不会匹配任何 shingles 。只是索引 bigrams 是不够的；我们仍然需要 unigrams ，但可以将匹配 bigrams 作为增加相关度评分的信号。

将它们分开保存在能被独立查询的字段会更清晰：
```
PUT /shingle_index
{
  "settings": {
    "number_of_shards": 1,
    "analysis": {
      "filter": {
        "my_shingle_filter": {
          "type": "shingle",
          "min_shingle_size": 2,
          "max_shingle_size": 2,
          "output_unigrams": false
        }
      },
      "analyzer": {
        "my_shingle_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_shingle_filter"
          ]
        }
      }
    }
  }
}
```
用 analyze API 测试下分析器：
```
GET /shingle_index/_analyze
{
  "analyzer": "my_shingle_analyzer",
  "text": "Sue ate the alligator"
}
```
得到了 3 个词项：
```
["sue ate", "ate the", "the alligator"]
```
将unigrams和bigrams分开索引更清晰，所以title字段将创建成一个多字段：
```
PUT /shingle_index/_mapping/
{
  "properties": {
    "title": {
      "type": "text",
      "fields": {
        "shingles": {
          "type": "text",
          "analyzer": "my_shingle_analyzer"
        }
      }
    }
  }
}
```
索引以下示例文档：
```
POST /shingle_index/_doc/_bulk
{"index":{"_id":1}}
{"title":"Sue ate the alligator"}
{"index":{"_id":2}}
{"title":"The alligator ate Sue"}
{"index":{"_id":3}}
{"title":"Sue never goes anywhere without her alligator skin purse"}
```
使用The hungry alligator ate Sue 进行简单 match 查询：
```

GET /shingle_index/_doc/_search
{
  "query": {
    "match": {
      "title": "the hungry alligator ate sue"
    }
  }
}
```
这个查询返回了所有的三个文档， 但是注意文档 1 和 2 有相同的相关度评分因为他们包含了相同的单词：
```
{
    "total": {
        "value": 3,
        "relation": "eq"
    },
    "max_score": 1.3721708,
    "hits": [
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "1",
            "_score": 1.3721708,
            "_source": {
                "title": "Sue ate the alligator"
            }
        },
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "2",
            "_score": 1.3721708,
            "_source": {
                "title": "The alligator ate Sue"
            }
        },
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "3",
            "_score": 0.2152618,
            "_source": {
                "title": "Sue never goes anywhere without her alligator skin purse"
            }
        }
    ]
}
```
现在在查询里添加 shingles 字段，为了提高相关度评分，我们仍然需要将基本title字段包含到查询中：
```
GET /shingle_index/_doc/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "title": "the hungry alligator ate sue"
        }
      },
      "should": {
        "match": {
          "title.shingles": "the hungry alligator ate sue"
        }
      }
    }
  }
}
```
仍然匹配到了所有的 3 个文档， 但是文档 2 现在排到了第一名因为它匹配了 shingled 词项 ate sue：
```
{
    "total": {
        "value": 3,
        "relation": "eq"
    },
    "max_score": 3.6694741,
    "hits": [
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "2",
            "_score": 3.6694741,
            "_source": {
                "title": "The alligator ate Sue"
            }
        },
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "1",
            "_score": 1.3721708,
            "_source": {
                "title": "Sue ate the alligator"
            }
        },
        {
            "_index": "shingle_index",
            "_type": "_doc",
            "_id": "3",
            "_score": 0.2152618,
            "_source": {
                "title": "Sue never goes anywhere without her alligator skin purse"
            }
        }
    ]
}
```
## 分析