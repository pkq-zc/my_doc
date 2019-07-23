# Get Api

## 通过ID获取document

``` http
GET twitter/_doc/1
```

上面的请求代表从```twitter```索引中获取```id```为1的json格式文档.响应结果如下:

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "user" : "tomc",
    "post_date" : "2019-01-01 15:30:00",
    "message" : "the first elasticsearch index"
  }
}
```

- ```_index```代表索引的名称
- ```_found```代表是否找到
- ```_source```里面的内容就是document

还可以通过以下的方式判断文档是否存在

``` http
HEAD twitter/_doc/1
```

响应结果如下:````200 - OK``

## 实时

在默认情况下,get API是实时的,并且不受refresh影响(当数据对搜索可见时),如果数据已更新,还未刷新,使用get API时将执行一个refresh操作,使文档可见,同时还将使上次刷新以来更改的文档可见.为了禁用realtime,可以将realtime参数设置为false.关于什么是refresh,请参照官网:[refresh](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-refresh.html "refresh")

## source filtering

默认情况下,get操作返回```_source```字段的所有内容,除非使用了```stored_fields```参数或者禁用了```_source```字段.可以使用```_source```参数关闭返回的字段.例如:

``` http
GET twitter/_doc/1?_source=false
```

响应结果如下:

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true
}
```

从响应结果可以看出,```_source```没有返回.
有些情况下,document字段太多,我们并不需要返回所有的字段.我们可以使用```_source_includes```和```_source_excludes```来指定哪些字段返回,哪些字段不返回.通过上述方法,当文档字段特别多时,可以大大的减少网络开销,提高系统响应速度.例如原始文档如下:

``` jsson
{
  "user":"jack",
  "user_email":"jack@163.com",
  "message":"this is test message",
  "post":"2019-04-01 09:00:01"
}
```

- 实例1:返回user,user_email,message字段,请求如下

``` http
GET twitter/_doc/2?_source_includes=user*,message
```

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "_seq_no" : 3,
  "_primary_term" : 2,
  "found" : true,
  "_source" : {
    "user_email" : "jack@163.com",
    "message" : "this is test message",
    "user" : "jack"
  }
}
```

- 实例2:不返回user字段,请求如下

``` http
GET twitter/_doc/2?_source_excludes=user
```

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "_seq_no" : 3,
  "_primary_term" : 2,
  "found" : true,
  "_source" : {
    "user_email" : "jack@163.com",
    "post" : "2019-04-01 09:00:01",
    "message" : "this is test message"
  }
}
```

## 只返回_source

可以通过```{index}/_source/{id}```只返回```_source```里面的字段,同时该适用于```source filtering```.

``` http
GET twitter/_source/1
```

响应结果如下:

``` json
{
  "user" : "tomc",
  "post_date" : "2019-01-01 15:30:00",
  "message" : "the first elasticsearch index"
}
```

同时使用```source filter```

``` http
GET twitter/_source/1?_source_excludes=post_date
```

响应结果如下:

``` json
{
  "message" : "the first elasticsearch index",
  "user" : "tomc"
}

```

## routing

在插入数据时,可以使用```routing```控制路由.在使用get api时,也应该提供routing,如果不提供routing,可能会导致无法找到相应数据.有的时候,不使用routing也可以查询到数据,这个主要是因为primary shard的设置导致的.如果primary shard数量比较小时,不设置routing也能搜索到.请看下面例子:

``` http
PUT order
{
  "settings": {
    "number_of_shards": 16,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "order_no":{"type":"keyword"},
      "order_price":{"type":"double"}
    }
  }
}
```

插入两条数据:

``` http
POST order/_doc/1?routing=java
{
  "order_no":"java",
  "order_price":12.91
}
```

``` http
POST order/_doc/2?routing=elasticsearch
{
  "order_no":"elasticsearch",
  "order_price":50.91
}
```

搜索1:不是设置```routing```

``` http
GET order/_doc/1
```

响应结果1:

``` json
{
  "_index" : "order",
  "_type" : "_doc",
  "_id" : "1",
  "found" : false
}

```

搜索2:设置错误的```routing```

``` http
GET order/_doc/1?routing=zxc
```

响应结果2:

``` json
{
  "_index" : "order",
  "_type" : "_doc",
  "_id" : "1",
  "found" : false
}
```

搜索3:设置正确的```routing```

``` http
GET order/_doc/1?routing=java
```

响应结果3:

``` http
{
  "_index" : "order",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "_routing" : "java",
  "found" : true,
  "_source" : {
    "order_no" : "java",
    "order_price" : 12.91
  }
}
```
