# Bitmap的原理

## 问题

在网上有一道面试题大概意思就是:```存在十亿个数字,现在需要知道哪些数字没有出现在里面.```这个问题你可能觉得一个for循环就能解决,但是实际上你可能忽略了一个问题,那就是10亿个数字需要占据的内存空间.例如在java中要存下10亿个数字需要多大空间呢?假设使用int存储,一个int占32个位也就是4个字节.存放10亿个数字则需要:```10亿*4/1024/1024/1024 = 4G左右```.面对上面的问题我们肯定不能使用这种方式存储.而使用我们的Bitmap就能很好的解决上面的问题.

## bitmap

Bitmapi简单的说就是使用bit(位)来存储状态,适合用来存储大规模的数据,但是状态又不是很多的情况.举例来说,如果我现在有```1,3,4,2,7```这五个数字,如果我们使用5个int来存储则需要20个字节的空间.但是我现在使用1字节就能存储上面的五个数字.如下图所示,1个字节可以用8位表示,1位有0和1两种值分别用来表示该位上是否有值,而位的索引用来表示数字(图中以从左到右的顺序计算).  
![bitmap存储数据原理.jpg](https://i.loli.net/2020/11/24/HitE2DmNXy3w9do.png)  
使用bitmap我们存储10个数字需要的空间为:```10亿/8/1024/1024 = 100MB左右```.相比于之前的方式,大大降低需要的空间.而且数据还有以下几个特点:  

- 数据是有序的
- 数据已经被去重了

而它的缺点同样也比较明显.它的一个位只能存储一个数据,对于重复的数据无法记录.如果数据分布很稀疏,会造成空间的浪费.例如,如果现在只有{1,3,1000000000}这个三个数据,它有很多空间会被浪费掉.

## 应用

BitMap在java中提供了BitSet的实现,同时在我们平常使用的Redis中也有实现.它很适合用来做大量的用户统计.例如统计登录人数,签到,在线等等.这些需求都可以通过redis的bit相关指令很简单的就能实现.  
例如,统计网站的在线人数:  

```shell
#记录用户2020年11月4号ID为1的用户登录
127.0.0.1:6379> setbit 20201124 1 1
(integer) 0
127.0.0.1:6379> setbit 20201124 2 1
(integer) 0
127.0.0.1:6379> setbit 20201124 3 1
(integer) 0
127.0.0.1:6379> setbit 20201124 4 1
(integer) 0
# 统计2020年11月24日 登录用户的总数
127.0.0.1:6379> bitcount 20201124
(integer) 4
```

上面的方式不仅简单,而且就算用户数量很大也能又快又好的完成.
