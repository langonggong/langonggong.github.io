---
title: Elasticsearch常用搜索
description: Elasticsearch常用搜索
date: 2018-08-19 12:41:41
keywords: Elasticsearch
categories : [Elasticsearch]
tags : [Elasticsearch, DSL]
comments: true
---

## 多字段查找

`multi_match`		

以下表达式等价于`field1 = value1 or field2 = value1`

```
GET index/type/_search
{
  "query": {
    "multi_match": {
      "query": "value1",
      "fields": ["field1","field2"]
    }
  }
  , "_source": ["field1","field2"]
}
```

## 布尔查询

`must`等价于`and`,`must_not`等价于`not`,	`should`等价于`or`		

以下表达式等价于`( field1 = value1 or field2 = value2 ) and field3 != value3`

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "bool": {
      "must": [
        {
          "bool": {
            "should": [
              {
                "match": {
                  "field1": "value1"
                }
              },
              {
                "match": {
                  "field2": "value2"
                }
              }
            ]
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "field3": "value3"
          }
        }
      ]
    }
  },
  "_source": [
    "field1","field2","field3"
  ]
}
```

## 模糊查询

`fuzziness`
	
[参考链接](https://www.elastic.co/guide/cn/elasticsearch/guide/current/languages.html)

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "city": {
              "query": "Hoiliste",
              "fuzziness": "auto"
            }
          }
        }
      ]
    }
  },
  "_source": [
    "city"
  ]
}
```

## 通配符查询

`*`匹配零个或多个字符

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "bool": {
      "must": [
        {
          "wildcard": {
            "field1": {
              "value": "wu*"
            }
          }
        }
      ]
    }
  },
  "_source": [
    "field1"
  ]
}
```

## 正则查询

`regexp `

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "regexp":{
      "field1":"[a-zA-Z]+[0-9]*"
    }
  },
  "_source": [
    "field1"
  ]
}
```

## 范围查询

`range`

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "range": {
      "field1": {
        "from": "2018-04-16 05:22:39",
        "to": "2018-06-23 05:22:39",
        "include_lower": true,
        "include_upper": true
      }
    }
  },
  "_source": [
    "field1"
  ]
}
```

## 排序

`sort`

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "bool": {}
  },
  "sort": [
    {
      "field1": {
        "order": "desc"
      }
    }
  ],
  "_source": [
    "field1"
  ]
}
```

## 多重过滤

`filter`

```
GET index/type/_search
{
  "size": 100,
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "must": {
              "term": {
                "field1": "value1"
              }
            }
          }
        },
        {
          "bool": {
            "must": {
              "term": {
                "field2": "value1"
              }
            }
          }
        }
      ]
    }
  }, "_source": ["cityTerm", "mlsOrgId"]
}
```

## 脚本查询
[参考链接](https://my.oschina.net/secisland/blog/683518)

`script`
支持Groovy、java等多种语言的脚本

```
{
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "filter": {
              "script": {
                "script": {
                  "inline": "doc['field1'].value - doc['field2'].value > 0"
                }
              }
            }
          }
        }
      ]
    }
  }
}

```