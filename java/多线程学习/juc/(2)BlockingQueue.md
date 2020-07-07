# (2)BlockingQueue

## Queue

Queue属于集合.它除了支持集合的基本操作,同时还支持插入,获取和检查存在操作.

|操作|抛出异常|返回特殊值|
|:---|:---|:---|
|插入|add(e)|offer(e)|
|移除|remove()|poll()|
|检查|element()|peek()|

## BlockingQueue

该接口继承```Queue```.相比于Queue,它还支持两个额外的操作:获取元素时等待元素不为空,存储元素时等待队列空间变得可用.

|操作|抛出异常|返回特殊值|
|:---|:---|:---|
|插入|put(e)|offer(e,time,unit)|
|移除|take()|poll(time,unit)|

需要注意的是,BlockingQueue不接受null元素.

## 主要实现类

### ArrayBlockingQueue

它是一个底层基于```数组```实现的```有界阻塞```队列.该队列按照```FIFO(先进先出)```顺序排列.同时该类还支持对等待的生产者线程和使用者线程进行顺序的可选公平性策略.但是如果设置为公平模式会降低吞吐量.

```java
public class App5 {
    public static void main(String[] args) throws InterruptedException {
        //创建队列
        final ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);
        //创建线程从队列中获取元素
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                Thread t = Thread.currentThread();
                while (true){
                    System.out.println(t.getName()+":准备从队列中获取元素");
                    try {
                        Integer data = queue.take();
                        System.out.println(t.getName()+":从队列中获取元素["+data+"]");
                        Thread.sleep(2000L);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread.setName("获取线程");
        thread.start();
        //主线程往队列添加元素
        Thread.sleep(2000L);
        for (int i = 0; i < 10; i++) {
            try {
                queue.put(i);
                System.out.println("添加数据:["+i+"]");
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

上面这段代码,一个线程往队列中添加数据另一个线程往队列中移除数据.因为移除元素的线程先执行,但是因为队列中的元素为空,所以移除元素线程就阻塞了.而当队列满了,添加元素的线程也会被阻塞,直到队列有足够的空间存储新的元素.这就是```阻塞```的意思.同时从队列中移除元素和放入元素的顺序是一致的,这就是我们所说的```FIFO```.初始化时,我们设置了队列的长度,所以说它是```有界```的.需要注意的是,我们只能在创建队列的时候设置它的长度,而这个长度后面我们将无法更改.

### LinkedBlockingQueue

它是一个底层基于```链表```实现的```长度不固定的阻塞```队列,因为它底层是通过链表去实现的.但是它在初始化时可以指定容量,防止过度扩充.如果没有指定容量,它最多可以存储```Integer.MAX_VALUE```个数据.该队列按照```FIFO(先进先出)```顺序排列.同样该队列也不知道插入null元素.

### PriorityBlockingQueue

一个```无界阻塞队列```,默认情况下按照```自然顺序排列```即从小到大排列.也支持在初始化时指定自定义的```Comparator```排列.需要注意的是,放入该队列中的元素必须实现```Comparable```接口或者在队列初始化时指定```Comparator```,否者将抛出```ClassCastException```.

```java
public class App2 {
    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<User> queue = new PriorityBlockingQueue<>(10, (o1, o2) -> {
            //按照age倒序排
            return o2.getAge() - o1.getAge();
        });

        //添加元素
        queue.put(new User("mac",18));
        queue.put(new User("tom",1));
        queue.put(new User("jack",19));
        queue.put(new User("rose",17));

        //移除队列数据
        while (true){
            User user = queue.take();
            System.out.println(user);
        }

    }
}

class User {
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

上面的代码打印结果如下:

```java
User{name='jack', age=19}
User{name='mac', age=18}
User{name='rose', age=17}
User{name='tom', age=1}
```

我们在创建队列时设置了自定义的```Comparator```,让元素按照从```age```从大到小排列.

### DelayQueue

DelayQueue是一个```无界阻塞```队列,只有延时期满了才能从中提取元素.该队列头部是```延时期满```后保存时间最长的元素.如果该队列的延时期都没满,则队列没有头部,即无法使用take或者poll移除未到期的元素.  

如果元素要放入```DelayQueue```,则该元素必须为```Delayed```实例.```Delayed```是一个接口,该接口主要实现两个方法:  

- ```getDelay(TimeUnit unit)```:返回剩余时间
- ```compareTo(Delayed o)```:比较此对象与指定对象的顺序。如果该对象小于,等于或大于指定对象,则分别返回负整数,零或正整数.需要注意的是```Delayed接口的实现类必须实现compareTo()方法，并且compareTo()方法必须提供与getDelay()方法一致的排序规则和顺序```  

因为延时这个特性,我们可以使用它来实现缓存的过期自动删除,订单过期自动关闭等等.

```java
public class App {
    public static void main(String[] args) throws InterruptedException {
        DelayQueue<CacheData> queue = new DelayQueue<>();

        //放入缓存数据
        queue.put(new CacheData("过期时间为[3]s的数据",3000L));
        queue.put(new CacheData("过期时间为[10]s的数据",10000L));
        queue.put(new CacheData("过期时间为[1]s的数据",1000L));
        queue.put(new CacheData("过期时间为[5]s的数据",5000L));
        queue.put(new CacheData("过期时间为[8]s的数据",8000L));

        //创建一个线程从缓存中移除过期数据
        Thread t = new Thread(() -> {
            while (true) {
                try {
                    //从队列头部移除过期元素
                    CacheData data = queue.take();
                    System.out.println(Thread.currentThread().getName()+":" + data);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t.setName("删除过期数据线程");
        //设置成守护线程,当主线程停止运行,该线程也将被取消掉程序结束运行
        t.setDaemon(true);
        t.start();

        Thread.sleep(11000L);
    }
}

/**
 * 缓存元素
 */
class CacheData implements Delayed{
    /**
     * 缓存数据
     */
    private Object data;
    /**
     * 过期时间
     */
    private Long ttl;

    public CacheData(Object data, Long ttl) {
        this.data = data;
        this.ttl = ttl + System.currentTimeMillis();
    }

    @Override
    public long getDelay(TimeUnit unit) {
        //获取剩余时间
        long time = ttl - System.currentTimeMillis();
        return unit.convert(time,TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o.getDelay(TimeUnit.MILLISECONDS));
    }

    @Override
    public String toString() {
        return "CacheData{" +
                "data=" + data +
                ", ttl=" + ttl +
                '}';
    }
}
```

打印结果如下:

```java
删除过期数据线程:CacheData{data=过期时间为[1]s的数据, ttl=1594092779233}
删除过期数据线程:CacheData{data=过期时间为[3]s的数据, ttl=1594092781230}
删除过期数据线程:CacheData{data=过期时间为[5]s的数据, ttl=1594092783233}
删除过期数据线程:CacheData{data=过期时间为[8]s的数据, ttl=1594092786233}
删除过期数据线程:CacheData{data=过期时间为[10]s的数据, ttl=1594092788230}
```

我们通过主线程添加数据到延时队列,另外创建一个守护线程用来移除队列中过期的元素.元素按照延迟时间由小到大移除被移除队列.

### SynchronousQueue

它是一个```没有容量的阻塞```队列.其中每一个插入必须等待另一个线程对应的移除操作,反之亦然.

```java
public class App3 {
    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue<Integer> queue = new SynchronousQueue<>();

        //线程往队列放入元素
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    try {
                        Thread.sleep(1000L);
                        queue.put(i);
                        System.out.println(Thread.currentThread().getName()+":放入元素:"+i);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        t1.setName("t1");
        t1.setDaemon(true);
        t1.start();

        // 线程从队列中获取数据
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    try {
                        Integer data = queue.take();
                        System.out.println(Thread.currentThread().getName()+":获取数据:"+data);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        t2.setName("t2");
        t2.setDaemon(false);
        t2.start();

        Thread.sleep(11000L);
    }
}
```

通过```Executors.newCachedThreadPool```创建线程池时,它使用的workQueue就是```SynchronousQueue```.源码如下:

```java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

## 总结

|名称|顺序|是否有界|特性|
|:---|:---|:---|:---|
|ArrayBlockingQueue|FIFO|有界|使用数组实现|
|LinkedBlockingQueue|FIFO|无界|使用链表实现|
|PriorityBlockingQueue|按照自然顺序或者自定义顺序|无界|根据优先级排序|
|DelayQueue|根据延迟时间排序|无界|只有元素到了过期时间才能取出|
|SynchronousQueue|-|-|没有长度|
