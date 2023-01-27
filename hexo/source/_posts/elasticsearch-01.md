---
title: ElasticSearch常见问题汇总(0)
date: 2021-04-24 19:45:46
tags:
 - ElasticSearch
categories:  ElasticSearch
description : ElasticSearch常见问题汇总
---

<!--more-->

## match/term等区别

**Elasticsearch中 match、match_phrase、query_string和term的区别**

首先建立一个索引，结果如下

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "name" : {"type": "keyword"}, //name是keyword类型，是不分词的
      "desc" : {"type": "text"}  // desc 是text类型，是会分词的
    }
  }
}
//插入一条数据
POST my_index/_doc/1
{
  "name" : "washing machin",
  "desc" : "I wrote an Elasticsearch article, which I think is quite easy to understand, and I hope it will be helpful for getting started"
}
```

**term查询**

term查询指的是完全匹配，即不进行分词器分析，文档中必须包含整个搜索的词汇，指的是 :

- 当查keyword类型时，由于keyword类型不会分词，所以需要完全匹配
- 当查textt类型时,text会分词，而term不分词，所以term查询的条件必须是text字段分词后的某一个

```json
GET my_index/_search
{
  "query": {
    "term": {
      "name": "washing"
    }
  }
}
//没值
GET my_index/_search
{
  "query": {
    "term": {
      "name": "washing machin"
    }
  }
}
//有值 查出来数据为空,因为name是keyword类型，name是不会被分词的，所以他的值就是washing machin，因为term是完全匹配，所以必须是查washing machin才有值

GET my_index/_search
{
  "query": {
    "match": {
      "name": "washing"
    }
  }
}
//没值，因为keyword是不会被分词的，where name = 'washing'。显然是查不到的

GET my_index/_search
{
  "query": {
    "match": {
      "name": "washing machin"
    }
  }
}
//有值，where name = 'washing machin'查得到

```

```json
GET my_index/_search
{
  "query": {
    "term": {
      "desc": "Elasticsearch"
    }
  }
}
//有值，desc是text类型，分词后包含Elasticsearch这个项。我们可以使用分词测试测试下是否包含Elasticsearch这个项

GET my_index/_analyze
{
  "analyzer": "standard",
  "text": "I wrote an Elasticsearch article, which I think is quite easy to understand, and I hope it will be helpful for getting started"
}

GET my_index/_search
{
  "query": {
    "term": {
      "desc": "an Elasticsearch article"
    }
  }
}
//如果查an Elasticsearch article 则没有值，因为term是完全匹配，相当于MySQL中的=

GET my_index/_search
{
  "query": {
    "match": {
      "desc": "an Elasticsearch article"
    }
  }
}
//如果使用match查询text类型，由于text是会被分词的，所以相当于where desc = 'an' or desc = 'Elasticsearch' or desc = 'article'.只要包含一个分词项就匹配
```

