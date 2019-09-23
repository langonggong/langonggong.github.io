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

### Synonym Token Filter
同义词过滤器

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

- Porter Stem Token Filter
- Stemmer Token Filter：包含了大多数词干提取器
- KStem Token Filter
- Snowball Token Filter
## 过滤

### Length Token Filter
长度过滤器，会移除token流中太长或太短的标记

### Stop Token Filter
停用词过滤器，会移除自定义或指定文件中的停用词

### Predicate Token Filter
根据脚本里面的表达式来判断是否过滤掉某个词条，例如使用standard分析器后的"What Flapdoodle"分隔成两个词条，只保留长度大于5的词条，则"What"将被删除

## 切割

### Word Delimiter Token Filter
分隔符过滤器，根据一定的规则将单词分割为多个子字符串，或者删除单词中的分隔符。例如：

- generate_word_parts
	"Power-Shot", "(Power,Shot)" -> "Power" "Shot"
- generate_number_parts
	将数字拆出来："500-42" -> "500" "42"
- catenate_words
	将单词去掉分隔符后合并："wi-fi" -> "wifi"
- 其他

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

###Shingle Token Filter（滑动窗口）
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
## 流程

### Conditional Token Filter
根据表达式条件决定是否使用某个其他的过滤器，例如根据一个词条中字符的个数来判断是否要使用Lowercase Token Filter过滤器

### Keyword Marker Token Filter
保护某些单词不被stemmers（词干提取器）处理，可以自定义单词列表或者指定文件路径

### Keyword Repeat Token Filter
将每个输入的token复制，生成keyword和non-keyword两个。一般用于交给词干提取器处理，然后去重处理

```
PUT /keyword_repeat_example
{
  "settings": {
    "analysis": {
      "analyzer": {
        "stemmed_and_unstemmed": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "keyword_repeat",
            "porter_stem",
            "unique_stem"
          ]
        }
      },
      "filter": {
        "unique_stem": {
          "type": "unique",
          "only_on_same_position": true
        }
      }
    }
  }
}
```
测试用例如下：
```
POST /keyword_repeat_example/_analyze
{
  "analyzer": "stemmed_and_unstemmed",
  "text": "I like cats"
}
```
结果如下：
```
{
  "tokens" : [
    {
      "token" : "i",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "like",
      "start_offset" : 2,
      "end_offset" : 6,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "cats",
      "start_offset" : 7,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "cat",
      "start_offset" : 7,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```