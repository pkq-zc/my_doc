# (1)什么是juc

```juc```是java中```java.util.concurrent```包的简称.这个包里面的东西就是```Doug Lea```写的,它主要包括```atomic```支持原子操作类相关代码,```locks```java中锁相关代码,还有其他并发容器相关代码.

## atomic

这个包里面提供了许多支持原子相关操作类的代码,例如:```AtomicBoolean```,```AtomicInteger```...等等.这些类就是通过```CAS```来提供原子操作支持的.

## locks

这个包主要提供了很多java中的锁.例如:```ReentrantLock```,```ReentrantReadWriteLock```...等等.这些类就是通过```AQS```来实现的.

## 其他

在```java.util.concurrent```下的其他类主要提供了并发容器相关类,例如:```ConcurrentHashMap```,```ConcurrentLinkedQueue```...等等相关类.还有线程池相关类,例如:```ThreadPoolExecutor```,```ScheduledThreadPoolExecutor```...等等.

## 学习建议

这个包对于并发编程特别重要,而且有很多代码不是特别好理解.所以在学习时,不要一上来就看源码看实现,因为你真的看不懂,大神当我没说过.首先第一步推荐先把常用的学会如何使用,然后再去看它是如何实现的.
