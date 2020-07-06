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
|检查|无|无|

需要注意的是,BlockingQueue不接受null元素.

## 主要实现类

### ArrayBlockingQueue

它是一个底层基于```数组```实现的```有界阻塞```队列.次队列按照```FIFO(先进先出)```顺序排列.同时该类还支持对等待的生产者线程和使用者线程进行顺序的可选公平性策略.但是如果设置为公平模式会降低吞吐量.

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
