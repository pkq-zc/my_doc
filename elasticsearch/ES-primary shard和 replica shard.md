# primary shard 和 replica shard

ES作为一个分布式系统,需要保证数据的安全性和容灾性.如果数据全部存在一个节点上(即一台服务器)上,如果服务器宕机或者硬盘坏了,那服务就不可用,数据就有可能丢失.为了保证系统的高可用和数据安全,ES通过shard机制来解决上述问题.

## primary shard

数据被写入时,可以会被写入到多个primary shard中的一个,而且只会是其中一个shard,不可能存在一条数据被写入到多个primary shard.每个shard都是一个独立的Lucene实例,关于shard的个数,可以在创建索引时,手动设置.这个值在设置完成之后,无法修改.可以通过参数```number_of_shards```设置.

## replica shard

replica shar就是primary shard 的备份.如果一个primary shard损坏或者暂停服务,数据并不会丢失,可以在primary shard对于的replica shar中找到.同时replica shard 和其对应的 primary shard永远不会分配在同一台机器上(一台机器,多个节点情况除外).因为分配在同一个节点上,如果在同一个节点上,那就没有意义了.replica shard 的个数也是可以自己手动设置的.这个数是可以修改的.通过```number_of_replicas```设置该值.

## 效果图

![primary_shard和replica_shard分布图.png](https://i.loli.net/2019/07/21/5d347fc69e6e278298.png)

如上图所示,primary shard为3,replica shard 为2,所有shard的总个数可以通过公式:
```shard_number = primary_shard + replica_shard * primary_shard```
