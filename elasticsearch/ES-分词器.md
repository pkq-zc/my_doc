# ES-分词器(Analyzer)

把输入的文本块按照一定的策略进行分解，并建立倒排索引。在Lucene的架构中，这个过程由分析器(analyzer)完成。

## 主要组成

- ```character filter```:接收原字符流，通过添加、删除或者替换操作改变原字符流。例如：去除文本中的html标签，或者将罗马数字转换成阿拉伯数字等。一个字符过滤器可以有```零个或者多个```。

- ```tokenizer```：简单的说就是将一整段文本拆分成一个个的词。例如拆分英文，通过空格能将句子拆分成一个个的词，但是对于中文来说，无法使用这种方式来实现。在一个分词器中,```有且只有一个```tokenizeer

- ```token filters```：将切分的单词添加、删除或者改变。例如将所有英文单词小写，或者将英文中的停词`a`删除等。在```token filters```中，不允许将```token(分出的词)```的```position```或者```offset```改变。同时，在一个分词器中，可以有零个或者多个```token filters```.

## 索引和搜索分词

文本分词会发生在两个地方：

- ```创建索引```:当索引文档字符类型为```text```时，在建立索引时将会对该字段进行分词。

- ```搜索```：当对一个```text```类型的字段进行全文检索时，会对用户输入的文本进行分词。

## 配置分词器

默认ES使用```standard analyzer```，如果默认的分词器无法符合你的要求，可以自己配置。

### 分词器测试

可以通过```_analyzer```API来测试分词的效果。

```
POST _analyze
{
  "analyzer": "standard",
  "text": "The quick brown fox"
}
```

响应结果如下：

```
{
  "tokens" : [
    {
      "token" : "the",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "quick",
      "start_offset" : 4,
      "end_offset" : 9,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "fox",
      "start_offset" : 16,
      "end_offset" : 19,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}

```

同时你也可以按照下面的规则组合使用：
- 0个或者多个```character filters```
- 一个```tokenizer```
- 0个或者多个```token filters```

```
POST _analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase"],
  "text": "The quick brown fox"
}
```


响应结果如下：

```
{
  "tokens" : [
    {
      "token" : "the",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "quick",
      "start_offset" : 4,
      "end_offset" : 9,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "fox",
      "start_offset" : 16,
      "end_offset" : 19,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}

```

与之前不同的是，它会将切分的词进行小写处理。这是因为我添加了一个```lowercase```的```token filter```，它会将分词的词进行小写处理。

我们还可以在创建索引前设置一个自定义的分词器：

```
PUT /my_index?pretty
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_folded": { 
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "my_text": {
        "type": "text",
        "analyzer": "std_folded" 
      }
    }
  }
}


GET /my_index/_analyze?pretty
{
  "analyzer": "std_folded", 
  "text":     "Is this déjà vu?"
}


GET /my_index/_analyze?pretty
{
  "field": "my_text", 
  "text":  "Is this déjà vu?"
}
```

上面操作我们自定义了一个分词器```std_folded```，它的```tokenizer```为```standard```，同时有两个```token filter```分别为：```lowercase```和```asiciifolding```。我们在定义mapping时，设置了一个字段名为```my_text```,它的类型为```text```，我们指定它使用的分词器为我们定义的```std_folded```.在分词测试中，我们获取的结果为：

```
{
  "tokens" : [
    {
      "token" : "is",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "this",
      "start_offset" : 3,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "deja",
      "start_offset" : 8,
      "end_offset" : 12,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "vu",
      "start_offset" : 13,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}

```

### 配置内置分词器

内置的分词器无需任何配置我们就可以使用。但是我们可以修改内置的部分选项修改它的行为。

```
DELETE my_index

PUT /my_index?pretty
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",
          "stopwords": "_english_"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "my_text": {
        "type":     "text",
        "analyzer": "standard", 
        "fields": {
          "english": {
            "type":     "text",
            "analyzer": "std_english" 
          }
        }
      }
    }
  }
}


POST /my_index/_analyze?pretty
{
  "field": "my_text", 
  "text": "The old brown cow"
}


POST /my_index/_analyze?pretty
{
  "field": "my_text.english", 
  "text": "The old brown cow"
}
```

上面的例子中，我们配置分词器```std_english```,它使用的分词器为```standard```分词器，他的停词列表设置为```_english_```.然后字段```my_text```使用的是```standard```分词器，而字段```my_text.english```使用的是我们配置的```std_english```.最后的分词测试结果如下：

```
{
  "tokens" : [
    {
      "token" : "the",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "old",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 8,
      "end_offset" : 13,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "cow",
      "start_offset" : 14,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}
```

```
{
  "tokens" : [
    {
      "token" : "old",
      "start_offset" : 4,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 8,
      "end_offset" : 13,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "cow",
      "start_offset" : 14,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}
```

结果1和2的区别为，结果2中的停词```The```被删除，而结果1中的并没有。这是因为```my_text.english```配置了停词。

### 创建自定义分词器

当内置的分词器无法满足需求时，可以创建```custom```类型的分词器。

- ```tokenizer```:内置或定制的tokenizer.(必须)
- ```char_filter```:内置或定制的char_filter(非必须)
- ```filter```:内置或定制的token filter(非必须)
- ```position_increment_gap```:当值为文本数组时，设置改值会在文本的中间插入假空隙。设置该属性，对与后面的查询会有影响。默认该值为100.

```
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer":{
          "type":"custom",
          "tokenizer":"standard",
          "char_filter":["html_strip"],
          "filter":["lowercase","asciifolding"]
        }
      }
    }
  }
}
```

上面的示例中定义了一个名为```my_custom_analyzer```的分词器，该分词器的```type```为```custom```，```tokenizer```为```standard```，```char_filter```为```hmtl_strip```,```filter```定义了两个分别为：```lowercase```和```asciifolding```。运行分词测试：

```
POST my_index/_analyze
{
  "text": "Is this <b>déjà vu</b>?",
  "analyzer": "my_custom_analyzer"
}
```

结果如下：

```
{
  "tokens" : [
    {
      "token" : "is",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "this",
      "start_offset" : 3,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "deja",
      "start_offset" : 11,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "vu",
      "start_offset" : 16,
      "end_offset" : 22,
      "type" : "<ALPHANUM>",
      "position" : 3
    }
  ]
}
```

## 指定分词器

分词器的使用地方有两个：
- 创建索引时
- 进行搜索时

### 创建索引时指定分词器

如果设置手动设置了分词器，ES将按照下面顺序来确定使用哪个分词器：
- 先判断字段是否有设置分词器，如果有，则使用字段属性上的分词器设置
- 如果设置了```analysis.analyzer.default```，则使用该设置的分词器
- 如果上面两个都未设置，则使用默认的```standard```分词器

#### 为字段指定分词器

```
PUT my_index
{
  "mappings": {
    "properties": {
      "title":{
        "type":"text",
        "analyzer": "whitespace"
      }
    }
  }
}
```

#### 设置索引默认分词器

```
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default":{
          "type":"simple"
        }
      }
    }
  }
}
```

### 搜索时如何确定分词器

在搜索时，通过下面参数依次检查搜索时使用的分词器：
- 搜索时指定```analyzer```参数
- 创建mapping时指定字段的```search_analyzer```属性
- 创建索引时指定```setting```的```analysis.analyzer.default_search```
- 查看创建索引时字段指定的```analyzer```属性

如果上面几种都未设置，则使用默认的```standard```分词器。

#### 搜索时指定analyzer查询参数

```
GET my_index/_search
{
  "query": {
    "match": {
      "message": {
        "query": "Quick foxes",
        "analyzer": "stop"
      }
    }
  }
}
```

#### 指定字段的seach_analyzer

```
PUT my_index
{
  "mappings": {
    "properties": {
      "title":{
        "type":"text",
        "analyzer": "whitespace",
        "search_analyzer": "simple"
      }
    }
  }
}
```

#### 指定索引的默认搜索分词器

```
PUT my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default":{
          "type":"simple"
        },
        "default_seach":{
          "type":"whitespace"
        }
      }
    }
  }
}
```

上面指定创建索引时使用的默认分词器为```simple```分词器，而搜索的默认分词器为```whitespace```分词器。
