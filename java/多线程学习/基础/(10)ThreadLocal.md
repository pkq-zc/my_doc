# (10)ThreadLocal

## 作用

ThreadLocal为每个使用该变量的线程提供独立的变量副本,让每一个线程都可以独立地改变自己的副本,而不会影响其它线程所对应的副本.简单来说就是两点:

- 各自线程有各自的副本,且该副本只有当前的线程能使用
- 基于第一点,那也就不存在多线程共享的问题了,也不存在线程同步等.

```java
public class App2 {
    private static ThreadLocal<String> local = new ThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = () -> {
            String setName = Thread.currentThread().getName();
            local.set(setName);
            String getName = local.get();
            System.out.println(Thread.currentThread().getName() + " : getName = " + getName);
        };

        new Thread(runnable,"t1").start();
        new Thread(runnable,"t2").start();
        new Thread(runnable,"t3").start();
        new Thread(runnable,"t4").start();

        Thread.sleep(1000L);
    }
}
```

上面的代码,最后结果会对应的线程会打印出自己对应的线程名称.多个线程自己操作自己的变量互不干扰,并不会因为在多线程下而导致数据不安全.

## 主要方法

### 创建

``` java
ThreadLocal<Integer> local1 = ThreadLocal.withInitial(() -> 0);
ThreadLocal<Integer> local2 = new ThreadLocal<>();
```

创建使用就这两种方式,这两种方式的区别在于第一种有一个初始的默认值,在你没有放入值之前,调用get()能获取到一个初始值,第二种则会返回null.

### 使用

``` java
        //get
        Integer i = local1.get();
        //set
        local1.set(2);
        //remove
        local1.remove();
```

get()从副本中获取一个值,如果没有值,返回null;set(T t)往副本中放入一个值;remove()清空副本中的值

## 基本原理

### 基本数据结构

在Thread的内部存在一个变量threadLocals,它的类型是ThreadLocal.ThreadLocalMap  
![ThreadLocal内部的threadLocals.png](https://i.loli.net/2019/11/27/6vGc7jzQSfWxeHo.png)
而在ThreadLocalMap内部,它主要是通过一个数组table来存储数据,它的数据都被包装成Entry类型的数据存在table数组里面.同时它提供一系列数组扩容,数据查找,删除等方法.而Entry是一个使用ThreadLocal作为key,使用Object作为value的结构.  
![ThreadLocalMap内部使用table数组存储数字.jpg](https://i.loli.net/2019/11/27/OAmaRiU6nIKw8b5.png)  
![Entry结构.jpg](https://i.loli.net/2019/11/27/54copRbm1qtMZsh.png)

### 主要方法实现

当调用get方法时,首先是获取到Thread的副本threadLocals.获取到的就是一个ThreadLocalMap,然后调用ThreadLocalMap的查找方法获取,通过ThreadLocal自身作为Key获取到对应的值.  
![set方法主要实现.jpg](https://i.loli.net/2019/11/27/knM5ef4ChyKEDvB.png)  
![获取的就是Thread的ThreadLocalMap.jpg](https://i.loli.net/2019/11/27/biotdcz3yIZJau9.png)  

了解了get方法,那么set和remove大致都类似  
![remove方法主要实现.jpg](https://i.loli.net/2019/11/27/lwXQWquNsDeg5pV.png)  
![get方法主要实现.jpg](https://i.loli.net/2019/11/27/6rFVUfSClJPqynu.png)  
> ```总结来说,其实ThreadLocal就是提供了一个操作Thread内部副本的入口.数据最后存在的并不是ThreadLocal内部,而是存在Thread自身,所以它并不会导致多线程共享变量的不安全.```

## 需要注意的点

在使用时,也有几点需要值得注意.如果不注意,可能导致业务数据异常,也可能导致内存泄漏

### 业务异常

``` java
public class App4 {
    private static ThreadLocal<List<Integer>> local = ThreadLocal.withInitial(LinkedList::new);
    public static void main(String[] args) {
        Runnable runnable = () -> {
            int i = new Random().nextInt(1000);
            local.get().add(i);
            //System.out.println(Thread.currentThread().getName()+" : "+local.get());
            System.out.println(local.get());
        };

        ExecutorService pool = Executors.newFixedThreadPool(2);
        for (int i = 0; i < 10; i++) {
            pool.execute(runnable);
        }

        pool.shutdown();

    }
}
```

上面代码的结果如下:

```java
[293]
[892]
[293, 303]
[892, 982]
[293, 303, 565]
[892, 982, 88]
[293, 303, 565, 131]
[892, 982, 88, 244]
[293, 303, 565, 131, 587]
[892, 982, 88, 244, 919]
```

为什么会导致该结果产生呢?我们使用的是```Executors.newFixedThreadPool```,它产生的线程在线程池中是固定的.每次执行的都是两个线程中的其中一个,线程执行完之后并没有被结束,而是在线程池中缓存,等待执行下一次任务.正是因为如此,该线程上一次产生了一个随机数存在列表中,在任务执行完之后没有清空副本中的值,这就导致了业务数据的异常.所以在使用完之后,最好调用```remove```清空其中的值,避免引起业务数据异常.

### 内存泄漏

```ThreadLocalMap```使用的是```ThreadLocal```弱引用([什么是弱引用?](https://www.jianshu.com/p/43f80b057b97 "什么是弱引用"))作为key.如果ThreadLocal在外部没有一个强引用,那么在系统的下次GC时,将会被回收.这样一来,在```ThreadLocalMap```内部就会出现key为null的```Entry```.如果当前线程一直没有结束,那么存在的强引用将导致这些内存不会被GC.所以在使用完ThreadLocal之后,一定要调用它的remove方法,将不再使用的值清空.
