# DELETE API

## 根据ID删除文档

格式如下:

``` http
DELETE {index}/_doc/{id}
```

如果删除创建索引数据时,指定了```routing```的值,在删除时,也需要指定```routing```的值,否者可能导致以下结果:

``` json
{
  "_index" : "order",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "not_found",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```
