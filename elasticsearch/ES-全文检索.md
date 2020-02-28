# ES全文检索(full text query)

> 本文所使用的ES版本为```7.4```.在了解本文前,需要知道什么是[query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl.html).同时已经了解是什么[analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis.html).官网介绍的```Analyzer```比较多,如果只想粗略了解什么是```Analyzer```,可以参考本人之前写的[ES之分析器Analyzer](https://www.jianshu.com/p/bebea42b5040).

## 1.创建测试数据

``` http
DELETE test_index

PUT test_index

PUT test_index/_mapping
{
  "properties":{
    "content":{
      "type":"text"
    }
  }
}

POST test_index/_bulk
{"index":{"_id":"1"}}
{"content":"java is my favorite language"}
{"index":{"_id":"2"}}
{"content":"Go is my favorite language"}
{"index":{"_id":"3"}}
{"content":"java language is very good"}
```

## 2.[match query](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query.html)

```match_query```在查询之前,会先对查询的文本进行分词.例如下面例子:

```http
POST test_index/_search
{
  "query": {
    "match": {
      "content": "java language"
    }
  }
}
```

在查询前,会先将文本分词.因为我们没有设置分词器,es将会使用```standard```分词器将查询语句```java language```拆分成两个词```java```和```language```.它将在索引中找到有这两个词的数据返回.查询结果如下所示:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.60353506,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.60353506,
        "_source" : {
          "content" : "java is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "3",
        "_scor    e" : 0.60353506,
        "_source" : {
          "content" : "java language is very good"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.13353139,
        "_source" : {
          "content" : "Go is my favorite language"
        }
      }
    ]
  }
}
```

从结果可以看出,所有结果都已经全部查询出来.而且还根据相似度打了分.同时它还支持几个常用的参数.  

- ```operator```:该值默认为```OR```.意思就是查询文本被分词之后的,他们之间只要有一个存在,那么就会被查询出来.如果改成```AND```,意思就是被分词之后的词,需要全部出现才能被查询出来.

```http
POST test_index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "java language",
        "operator": "and"
      }
    }
  }
}
```

查询结果如下:

```json
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.60353506,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.60353506,
        "_source" : {
          "content" : "java is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.60353506,
        "_source" : {
          "content" : "java language is very good"
        }
      }
    ]
  }
}

```

与之前的结果比较,可以发现```Go is my favorite language```没有出现,因为它只满足```language```.

- ```minimum_should_match```.与```operator```类似,该值代表最小需要满足多少个条件.

```http
POST test_index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "java language so good",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

```java language so good```将会被```standard```分词器分为四个词,那么最小满足```75%```说明至少需要满足三个词才能被查询出来.所以结果如下:

``` json
{
  "tokens" : [
    {
      "token" : "java",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "language",
      "start_offset" : 5,
      "end_offset" : 13,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "so",
      "start_offset" : 14,
      "end_offset" : 16,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "good",
      "start_offset" : 17,
      "end_offset" : 21,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}
```

- ```fuzziness```:例如我们在查询```java```时,可能因为手误导致```java```写成了```jave```.那么查询结果将导致一个都无法匹配.我们可以通过该参数纠正错误,正确的查询出结果.

```http
POST test_index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "jave"
      }
    }
  }
}

POST test_index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "jave",
        "fuzziness": 1
      }
    }
  }
}

```

上面两个查询,第一个无法查找含有```java```的内容,但是查询二因为设置了```fuzziness```所以可以正确的匹配到.

## 3.[match_bool_prefix](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-bool-prefix-query.html)

该查询内部使用```analyzer```将查询文本分词,然后基于分词的内容进行bool query,除了最后一个使用prefix查询,其他都是term query.

```http
POST test_index/_search
{
  "query": {
    "match_bool_prefix":{
      "content":"java is m"
    }
  }
}
```

上面的查询可以解释为下面的查询:

```http
POST test_index/_search
{
  "query": {
    "bool": {
      "should": [
        {"term":{"content": "java"}},
        {"term":{"content": "is"}},
        {"prefix":{"content": "m"}}
      ]
    }
  }
}
```

这两个查询时等价的,最后的查询结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.603535,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.603535,
        "_source" : {
          "content" : "java is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.1335313,
        "_source" : {
          "content" : "Go is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.60353506,
        "_source" : {
          "content" : "java language is very good"
        }
      }
    ]
  }
}
```

## 4.[match_phrase](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query-phrase.html)

match_phrase查询将分析文本，并从分析的文本中创建短语查询.

```http
POST test_index/_search
{
  "query": {
    "match_phrase": {
      "content": "java is"
    }
  }
}
```

例如上面的查询,它会将```java is```做为一个短语查询,它的查询结果为:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.60353506,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.60353506,
        "_source" : {
          "content" : "java is my favorite language"
        }
      }
    ]
  }
}
```

上面的结果中,只有```java is my favorite language```被查询出来了,它与查询条件匹配,因为```java```和```is```之间是仅仅挨着的,不需要调换他们之间的位置.  

我们再添加一条测试语句,如下:

```http
POST test_index/_doc/4
{
  "content":"java and go is program language"
}
```

如果我们想让```java and go is program language```和```java language is very good```都能被查询出来呢?那么我们可以使用slop查询来改变.例如:

```http
POST test_index/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "java is",
        "slop": 2
      }
    }
  }
}
```

使用查询中,我们设置slop为2.```java language is very good```只需要移动1次便可以匹配到```java is```,而```java and go is program language```则需要两次.查询结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.471215,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.471215,
        "_source" : {
          "content" : "java is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.30669597,
        "_source" : {
          "content" : "java language is very good"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 0.20387812,
        "_source" : {
          "content" : "java and go is program language"
        }
      }
    ]
  }
}
```

如果我们将slop设置为1,则```java and go is program language```将无法满足条件无法被查询出来.我们再添加一条语句,如下:

```http
POST test_index/_doc/5
{
  "content":"my favorite language is java"
}
```

我们还是使用上面的查询,```my favorite language is java```还是可以被我们查询出来.需要注意的是,调换位置,slop为2,而不是1.

## 5.[match_phrase_prefix](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query-phrase-prefix.html)

返回包含所提供文本单词的文档,顺序与提供的顺序相同.所提供的文本的最后一个术语被视为前缀,与以该术语开头的任何单词相匹配.  
它与```match_phrase```最大的区别在于它多了一个```prefix```匹配.

```http
POST test_index/_search
{
  "query": {
    "match_phrase_prefix": {
      "content": "my favorite l"
    }
  }
}
```

上面的例子中,匹配短语为```my favorite```,匹配的前缀为```l```.现在再添加一个文档.

```http
POST test_index/_doc/6
{
  "content":"My favorite food and language are potatoes and Chinese "
}
```

查询条件如下:

```http
POST test_index/_search
{
  "query": {
    "match_phrase_prefix": {
      "content": {
        "query": "my favorite l",
        "slop":2
      }
    }
  }
}
```

查询结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 1.0172215,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0172215,
        "_source" : {
          "content" : "java is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0172215,
        "_source" : {
          "content" : "Go is my favorite language"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 1.0172215,
        "_source" : {
          "content" : "my favorite language is java"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "6",
        "_score" : 0.34737897,
        "_source" : {
          "content" : "My favorite food and language are potatoes and Chinese "
        }
      }
    ]
  }
}
```

可以发现,我们新添加的文档也可以被查询到.因为```match_phrase_prefix```同样也支持```slop```参数.使用```match_phrase_prefix```会消耗大量资源.因为```l```可能会匹配到成千上万的term.所以官方提供了一个参数```max_expansions```限制了扩展匹配的个数.默认该值为50.需要特别注意的一点是,不能以最后返回的结果作为```max_expansions```这个参数是否生效的验证.

## 6.[mutil_match](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-multi-match-query.html)

```multi_match```以```match```为基础,允许多个字段同时查询.例如下面例子插入测试数据:

```http
PUT test_index2

POST test_index2/_bulk
{"index":{"_id":"1"}}
{"title":"big data","content":"we can use java process big data"}
{"index":{"_id":"2"}}
{"title":"what is scala","content":"scala run in jvm and used in process big data"}
{"index":{"_id":"3"}}
{"title":"favorite language","content":"java and Go is my favorite language"}
{"index":{"_id":"4"}}
{"title":"About java","content":"java is language and run jvm"}
{"index":{"_id":"5"}}
{"title":"what is jvm ?","content":"Java Virtual Machine"}
```

然后查询```title```或```content```里面存在```java```的记录.

```http
POST test_index2/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["title","content"]
    }
  }
}
```

查询结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 1.4877305,
    "hits" : [
      {
        "_index" : "test_index2",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.4877305,
        "_source" : {
          "title" : "About java",
          "content" : "java is language and run jvm"
        }
      },
      {
        "_index" : "test_index2",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 0.37031418,
        "_source" : {
          "title" : "what is jvm ?",
          "content" : "Java Virtual Machine"
        }
      },
      {
        "_index" : "test_index2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.28072202,
        "_source" : {
          "title" : "big data",
          "content" : "we can use java process big data"
        }
      },
      {
        "_index" : "test_index2",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 0.28072202,
        "_source" : {
          "title" : "favorite language",
          "content" : "java and Go is my favorite language"
        }
      }
    ]
  }
}
```

### multi_match查询的类型

这个概念对于```multi_match```来说很重要.默认```multi_match```的类型为```best_fields```.例如上面的查询,完整的样子应该如下所示:

```http
POST test_index2/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["title","content"],
      "type": "best_fields",
    }
  }
}
```

除此之外还有另外几种:```most_fields```,```cross_fields```,```phrase```,```phrase_prefix```,```bool_prefix```.

#### best_fields

```best_fields```指的就是搜索结果中应该返回某一个字段匹配到了最多的关键词的文档.要搞懂```best_fields```首先要搞懂什么是```dis_max```.它的中文意思就是分离最大化查询的意思.  

- 创建示例数据:

```http
PUT test_index3

POST test_index3/_bulk
{"index":{"_id":"1"}}
{"title":"I like chinese food","content":"my favorite food is rice"}
{"index":{"_id":"2"}}
{"title":"food is very import","content":"to many people don't have food"}
```

- 查询

```http
POST test_index3/_search
{
  "query": {
    "match": {
      "title": "chinese food"
    }
  }
}

POST test_index3/_search
{
  "query": {
    "match": {
      "content": "chinese food"
    }
  }
}

POST test_index3/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match":{"title":"chinese food"}},
        {"match":{"content":"chinese food"}}
        ]
    }
  }
}
```

查询结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.87546873,
    "hits" : [
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.87546873,
        "_source" : {
          "title" : "I like chinese food",
          "content" : "my favorite food is rice"
        }
      },
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.18232156,
        "_source" : {
          "title" : "food is very import",
          "content" : "to many people don't have food"
        }
      }
    ]
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.18936403,
    "hits" : [
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.18936403,
        "_source" : {
          "title" : "I like chinese food",
          "content" : "my favorite food is rice"
        }
      },
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.17578416,
        "_source" : {
          "title" : "food is very import",
          "content" : "to many people don't have food"
        }
      }
    ]
  }
}
```

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.87546873,
    "hits" : [
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.87546873,
        "_source" : {
          "title" : "I like chinese food",
          "content" : "my favorite food is rice"
        }
      },
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.18232156,
        "_source" : {
          "title" : "food is very import",
          "content" : "to many people don't have food"
        }
      }
    ]
  }
}
```

注意观察结果的得分,最后的得分是```获取满足条件的最大得分```.而其他得分会被舍去,并不会计算到最后结果评分中去.但是有的时候我们希望其他匹配的条件也能为搜索贡献自己的分数时,我们可以通过设置```tie_breaker```来使其他得分按比例计算到最终得分中去.该值的取值范围在```0.0~1.0```之间.例如下面的例子:

```http
POST test_index3/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match":{"title":"chinese food"}},
        {"match":{"content":"chinese food"}}
        ],
        "tie_breaker": 1
    }
  }
}
```

最后的结果如下:

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0648328,
    "hits" : [
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0648328,
        "_source" : {
          "title" : "I like chinese food",
          "content" : "my favorite food is rice"
        }
      },
      {
        "_index" : "test_index3",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.35810572,
        "_source" : {
          "title" : "food is very import",
          "content" : "to many people don't have food"
        }
      }
    ]
  }
}
```

我们设置了```tie_breaker```为1,意思是将其他得分按```100%```的比例计算到最终的得分.从最后的结果和之前的结果对比可以看出该参数确实有效.  

理解了什么是```dis_max```了,就能很好理解什么是```best_fields```策略了.对于下面的查询可以理解为同等意思.

```http
POST test_index2/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["title","content"],
      "type": "best_fields"
    }
  }
}

POST test_index2/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match":{"title":"java"}},
        {"match":{"content":"java"}}
        ]
    }
  }
}
```

而这两个查询最后的结果也是一样的.

#### most_fields

```most_fields```这个与```best_fields```刚好相反,它会累计所有field的得分而不是取最高得分.```best_fields```就是搜索结果应该返回匹配了更多的字段的document优先返回回来.例如:

```http
POST test_index2/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["title","content"],
      "type": "most_fields"
    }
  }
}
```

它就等于下面这种查询:

```http
POST test_index2/_search
{
  "query": {
    "bool": {
      "should": [
        {"match":{"content": "java"}},
        {"match":{"title": "java"}}
      ]
    }
  }
}
```

#### phrase and phrase_prefix

这两种同```phrase query```和```phrase_prefix qeury```,不同的在于支持多字段同时匹配.需要注意的是它们默认的得分策略像```best_fields```只取最高分.例如下面的查询可以看做是相同的作用:

```http
POST test_index2/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["title","content"],
      "type": "phrase"
    }
  }
}

POST test_index2/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match_phrase":{"title":"java"}},
        {"match_phrase":{"content":"java"}}
        ]
    }
  }
}
```

#### cross_fields

插入示例数据:

```http
DELETE test_index4
PUT test_index4
POST test_index4/_bulk
{ "index": { "_id": "1"} }
{"first_name" : "Peter", "last_name" : "Smith"}
{ "index": { "_id": "2"} }
{"first_name" : "Smith", "last_name" : "Williams"}
{ "index": { "_id": "3"} }
{"first_name" : "Jack", "last_name" : "Ma"}
{ "index": { "_id": "4"} }
{"first_name" : "Robbin", "last_name" : "Li"}
{ "index": { "_id": "5"} }
{"first_name" : "Tonny", "last_name" : "Peter Smith"}
```

现在我们想一个全名里面含有```perter smith```的人.根据之前介绍的内容,你可能会写下面两种:

```http
POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name", "last_name"],
      "type": "best_fields"
    }
  }
}

POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name", "last_name"],
      "type": "most_fields"
    }
  }
}
```

这两种的返回结果中都含有这条数据:

```json
{
        "_index" : "test_index4",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.3862944,
        "_source" : {
          "first_name" : "Smith",
          "last_name" : "Williams"
        }
}
```

实际这条数据并不太符合我们的要求,因为里面没有```perter```.可能我们会修改上面的两个查询,增加```operator```提高他的精准度.

```http
POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name", "last_name"],
      "type": "best_fields",
      "operator": "and"
    }
  }
}

POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name", "last_name"],
      "type": "most_fields",
      "operator": "and"
    }
  }
}
```

但是上面这两个查询还是会有一个问题,那就是这条数据无法被我们找到:

```json
{
        "_index" : "test_index4",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.3862944,
        "_source" : {
          "first_name" : "Peter",
          "last_name" : "Smith"
        }
}
```

其实这条数据符合我们的要求,但是却无法被找到.如果想解决这个查询,我们可以在创建索引的时候使用```copy_to```,将```first_name```和```last_name```添加到一个新字段.例如下面这种:

```http
PUT test_index5

POST test_index5/_mapping
{
  "properties":{
    "first_name":{
      "type":"text",
      "copy_to":"full_name"
    },
    "last_name":{
      "type":"text",
      "copy_to":"full_name"
    },
    "full_name":{
      "type":"text"
    }
  }
}

POST test_index5/_bulk
{ "index": { "_id": "1"} }
{"first_name" : "Peter", "last_name" : "Smith"}
{ "index": { "_id": "2"} }
{"first_name" : "Smith", "last_name" : "Williams"}
{ "index": { "_id": "3"} }
{"first_name" : "Jack", "last_name" : "Ma"}
{ "index": { "_id": "4"} }
{"first_name" : "Robbin", "last_name" : "Li"}
{ "index": { "_id": "5"} }
{"first_name" : "Tonny", "last_name" : "Peter Smith"}

POST test_index5/_search
{
  "query": {
    "match": {
      "full_name": {
        "query": "Peter Smith",
        "operator": "and"
      }
    }
  }
}
```

上面这种固然很好,但是并不是很方便.我们可以使用```cross_fields```实现同样的功能.

```http
POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name","last_name"],
      "operator": "and",
      "type": "cross_fields"
    }
  }
}
```

```cross_fields```它会将多个```field```合并成一个```field```然后查询.这与建立一个新的索引保存多个```field```,然后再新字段上查询很相似.但是有点在于它不需要你修改```mapping```然后重建索引,而且他还能动态的设置字段的权重.例如下面这面这种方式:

```http
POST test_index4/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "fields": ["first_name^3","last_name"],
      "operator": "and",
      "type": "cross_fields"
    }
  }
}
```

通过```^数字```这种方式,我们可以动态的修改权重,从而影响得分.总体而言,```cross_fields```这种方式更像是```从多个字段中进行查询,但是却像在一个字段内查询这样```.

#### bool_prefix

该类型的评分行为类似于```most_fields```,但使用```match_bool_prefix```查询.
