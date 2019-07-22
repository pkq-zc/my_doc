# 如何创建索引

## 创建索引

``` http
PUT twitter/_doc/1
{
  "user":"tomc",
  "post_date":"2019-01-01 15:30:00",
  "message":"the first elasticsearch index"
}
```

创建一个名为```twitter```的索引,索引的```id```为1,下面为响应结果:

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

```_shard```中有许多节点:

- ```total```代表操作应该在多少个shard(分片)上执行,这里是2个.

- ```successful```代表多少个shard(分片)执行成功了.因为我只有一个es节点,primary shard和replica shard不能在一台机器上,所以只有primary shard 成功.如果一个操作成功,successful至少要大于等于1

## 自动创建索引

如果在你创建索引之前,索引不存在,ES会自动的为你的数据创建Mapping(类似数据库的表结构),也可以设置不允许自动创建表结果,修改```action.auto_create_index```即可,默认该值为```true```.

## 操作类型

该方法还可以接受一个```op_type```参数,指定操作类型,如果ID相同的数据已经存在,则会抛出异常,例如:

``` http
PUT twitter/_doc/1?op_type=create
{
  "user":"tomc",
  "post_date":"2019-01-01 15:30:00",
  "message":"the first elasticsearch index"
}
```

响应结果如下:

``` json
{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[1]: version conflict, document already exists (current version [1])",
        "index_uuid": "6VyZjkMzQfWIp0ByEkJgqQ",
        "shard": "0",
        "index": "twitter"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[1]: version conflict, document already exists (current version [1])",
    "index_uuid": "6VyZjkMzQfWIp0ByEkJgqQ",
    "shard": "0",
    "index": "twitter"
  },
  "status": 409
}
```

## 自动ID生成

在创建索引时,可以不设置ID,ES会自动生成ID.同时```op_type```将自动设置成```create```,此时需要用```POST```替换之前的```PUT```.例如:

``` http
POST twitter/_doc
{
  "user":"mac",
  "post_date":"2019-01-01 16:52:00",
  "message":"Today is a good day"
}
```

响应结果如下:

``` json
{
  "_index" : "twitter",
  "_type" : "_doc",
  "_id" : "i_fHFGwBQgY3x2bTEE0_",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
```

```_id```即是自动生成的ID

## 乐观锁并发控制

ES的是分布式系统.在对索引进行创建,删除,修改时,数据的新版本会被复制到其他节点,这个过程是异步和并发的,这也导致另外一个问题,请求到达目的地的顺序可能不会一致.所以使用了乐观锁来保证数据的安全和一致性.
响应结果中的```_seq_no```和```_primary_term```这两个字段就是用来实现乐观锁控制的.```version```这个字段,是在老版本用来实现乐观锁的,不过现在已经不用了.具体请参照官网:[Optimistic concurrency control](https://www.elastic.co/guide/en/elasticsearch/reference/current/optimistic-concurrency-control.html "Optimistic concurrency control")

## 路由

默认情况下,ES根据```_id```字段分片决定数据存在哪个分片上.可以通过在创建数据时,指定路由的字段,显示的控制.这个有利于某个值相同的数据落在同一个分片上,在查询阶段只需要查询一个分片,同时减少查询结果合并的开销,提高查询效率.还可以在设置```Mapping```时,指定```_routing```的值,但是如果该值为空,将会导致异常.详情可以参照官网:[_routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html "_routing")

## 移除Mapping types

在7.0之后的版本,不在支持```mapping type```.最主要是，存储在同一索引中具有很少或没有共同字段的不同实体会导致数据稀疏，并影响Lucene有效压缩文档的能力.详情见官网:[Removal of types](https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html "Removal of types")

## 超时

在索引的时候有可能primary shard不可用.默认情况下,最多等待1分钟然后响应错误.这个值我们可以自己设定.例子如下:

``` http
POST twitter/_doc?timeout=2m
{
  "user":"amber",
  "post":"2018-01-05 12:53:23",
  "message":"what's your name?"
}
```

上面的例子设置超时时间为2分钟,如果超过两分钟,primary shard还未操作成功,便会超时报错.

## 版本控制

```_version```响应结果中的该字段在低版本中是用来做occ(乐观锁的),不过该功能在6.7版本之后已经不再提倡使用.例如下面的请求:

``` http
POST twitter/_doc/1?version=1
{
  "user":"tomc2",
  "post_date":"2019-01-01 15:30:00",
  "message":"the first elasticsearch index"
}
```

响应结果如下:

``` json
{
  "error": {
    "root_cause": [
      {
        "type": "action_request_validation_exception",
        "reason": "Validation Failed: 1: internal versioning can not be used for optimistic concurrency control. Please use `if_seq_no` and `if_primary_term` instead;"
      }
    ],
    "type": "action_request_validation_exception",
    "reason": "Validation Failed: 1: internal versioning can not be used for optimistic concurrency control. Please use `if_seq_no` and `if_primary_term` instead;"
  },
  "status": 400
}
```

结果中提示,不需要再使用```_version```字段,改用```if_seq_no```和```if_primary_term```字段
