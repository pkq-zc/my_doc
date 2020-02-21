# ES-近义词查询

在现实生活中,我们搜索"马铃薯"时,百度能给我找到马铃薯,土豆,洋芋等相关信息.因为在它们都是同一个东西或者是同义词.在ES中使用全文搜索时,我们也能实现同样的功能.

## Synonym graph

ES内置提供 ```Synonym graph token filter````.我们可以使用它来实现我们我们上述功能.官方介绍:[Synonym graph](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/analysis-synonym-graph-tokenfilter.html)

- 自定义分词器

```http
PUT test_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_ik_max_word":{
          "type":"custom",
          "tokenizer":"ik_max_word",
          "filter":["graph_synonyms"]
        },
        "my_ik_smart":{
          "type":"custom",
          "tokenizer":"ik_smart",
          "filter":["graph_synonyms"]
        }
      },
      "filter": {
        "graph_synonyms":{
          "type":"synonym_graph",
          "synonyms_path":"analysis/synonym.txt"
        }
      }
    }
  }
}
```

- 设置mapping

```http
PUT test_index/_mapping
{
  "properties":{
    "content":{
      "type":"text",
      "analyzer":"my_ik_max_word"
    }
  }
}
```

上面需要注意的是我安装了ik分词器,然后自定义的分词器中的```filter```添加了```Synonym graph token filter```.```synonyms_path```的位置是以ES的```config```开始算的.所以我们需要创建```{es}/config/analysis/synonym.txt```文件来存放我们自定义的近义词表.近义词表中的格式如下:

```
土豆,洋芋 => 马铃薯
西红柿 => 番茄
```

接下来插入数据:

```http
POST test_index/_doc
{
  "content":"番茄炒蛋是一道菜"
}
POST test_index/_doc
{
  "content":"我喜欢吃土豆"
}
POST test_index/_doc
{
  "content":"马铃薯可以做薯片"
}
POST test_index/_doc
{
  "content":"我喜欢吃西红柿"
}
```

查询数据:

```http
GET test_index/_search
{
  "query": {
    "match": {
      "content": {
        "query": "洋芋",
        "analyzer": "my_ik_smart"
      }
    }
  }
}
```

结果如下:

```json
{
  "took" : 1,
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
    "max_score" : 0.7199211,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "03XBYXAB7AiRAPEjKnNG",
        "_score" : 0.7199211,
        "_source" : {
          "content" : "我喜欢吃土豆"
        }
      },
      {
        "_index" : "test_index",
        "_type" : "_doc",
        "_id" : "1HXBYXAB7AiRAPEjZHPB",
        "_score" : 0.7199211,
        "_source" : {
          "content" : "马铃薯可以做薯片"
        }
      }
    ]
  }
}
```

从搜索结果可以看出来,我们搜索的是```洋芋```,但是```我喜欢吃土豆```和```马铃薯可以做薯片```都被搜索出来了.因为我们设置它们是近义词.