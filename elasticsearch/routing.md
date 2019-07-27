# routing

## 思考

在ES中,我们可以给索引设置分片.但是有一个问题是,设置了n个分片,当我插入数据时,我插入的数据应该落在哪个分片里面呢?  
实际上,ES数据时落在哪个分片上,不是随机选取的.而是通过routing机制来实现的.  

## 如何计算数据应该落在哪个shard

官方提供的公式如下:
```
shard_num = hash(_routing) % num_primary_shards
```

- ```_routing```代表提供路由的字段.默认情况下,为文档的ID

- ```num_primary_shards```代表的为primary shard的个数,这个在每个索引类型创建之前就被设置了.可以手动设置,也可以让ES默认设置.因为ES版本不同,设置的默认值也不同.该值在第一次创建索引类型被设置完成之后,是无法无法修改的,这一点很重要,后面会说为什么该值被设置之后无法修改.

- ```shard_num```代表数据落在的shard的编号.

ES在决定document落在哪个分片上时,首先用路由字段通过```hash()```函数计算一个数字,然后拿这个数字和```primary shard```求余,获得的结果值在```0~(primary_shard - 1)```之间,这也解释了,为什么```primary shard```被设置之后无法修改.

## 解决自定义路由数据分布不均

由于自定义路由,很可能导致索引分布不均,导致大量数据集中在某一个shard上.为了解决这个问题,我们可以通过设置```routing_partition_size```来解决自定义路由不均的问题.该值的设置同样也是在创建索引的时候就被设置.路由的公式变为下面这个:

```
shard_num = (hash(_routing) + hash(_id) % routing_partition_size) % num_primary_shards
```

从上面的值公式可以看出,会使用```_id```字段再做一次计算,这样让文档的分布更加均匀.

## 官方文档

- [routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html "routing")

- [routing_partition_size](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#routing-partition-size "routing_partition_size")
