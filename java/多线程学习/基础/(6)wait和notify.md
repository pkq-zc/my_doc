# (6)wait和notify

## wait

该方法用来将```当前调用线程```置于WAITING(没有超时时间)或者TIMED_WAITING(有超时时间)状态,直到接到通知(notify),中断(interrupt)或者超时(timeOut).在调用wait()方法之前,线程必须已经获得该对象的对象级别锁,通常只能在同步代码块或者同步方法中调用,如果不是,将会报```IllegalMonitorStateException```异常.当该方法调用之后,当前线程就会释放所持的对象锁.

## notify和notifyAll

该方法用户通知那些可能等待在该对象上的对象锁的线程.如果是多个线程,调用该方法将随机唤醒一个(notify)或者多个(notifyAll)等待的线程.该方法同wait类似,也需要在同步代码块或者同步方法中调用,如果不是,同样将报```IllegalMonitorStateException```异常.当该方法被调用后,不会立马放掉对象锁,需要等待代码执行完,退出同步代码块或者同步方法才会释放锁.

## join的实现

之前在讲***Thread中常用的方法***时说活,join的实现是通过wait()实现的.

```java
    //该方法使用synchronized修饰
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            //判断线程是否存活
            while (isAlive()) {
                //调用wait方法
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

可以发现join是通过wait()来实现的,这也就是为什么join需要使用```synchronized```来修饰.分析下面一段join如何使用的代码如下:

```java
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+":执行完毕");
            }
        };
        System.out.println(Thread.currentThread().getName()+":t1.start()前执行");
        Thread t1 = new Thread(runnable);
        t1.setName("t1");
        t1.start();
        t1.join();
        System.out.println(Thread.currentThread().getName()+":t1.start()后执行");
```

结合上面join实现的源码,很多人不理解为什么main线程会进入等待.需要明白的是,谁调用某个实例的wait()方法,是调用者释放锁然后进入```WAITING```状态.t1只不过是一个普通的实例,并不是main线程调用t1的wait()导致t1线程进入```WAITING```状态.所以这就是为什么是main线程进入```WAITING```.  
既然main线程调用了wait()方法,那么main线程是被哪个线程调用notify唤醒的呢?我们自己的业务代码中并没有执行t1.notify().这个问题通过源码我也没有找到答案,最后求助知乎得到一个答案.大体来说就是jvm实现的线程,在线程彻底结束前会隐式的调用notifyAll().这部分代码在源码中是无法找到的.[知乎答案](https://www.zhihu.com/question/404139144)

## 锁池和等待池

在java对象中,每个对象都有一个与之唯一对应的内部锁(Monitor).java虚拟机会为每一个对象维护两个队列(暂且称之为队列),一个就是锁池,另外一个就是等待池.

- 锁池:存储等待获取objectX对应的内部锁的所有线程,你可以简单的理解这里面的线程都在等待获取这个对象上的锁.
- 等待池:于存储执行了objectX.wait()/wait(long)的线程,你可以简单的理解为这里面的线程都在等待唤醒.

``` java
public class Ch2 {
    public static void main(String[] args) throws InterruptedException {
        Object o = new Object();
        Runnable runnable = () -> {
            synchronized (o){
                try {
                    o.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                while (true){

                }
            }
        };

        Thread t1 = new Thread(runnable, "t1");
        Thread t2 = new Thread(runnable, "t2");
        Thread t3 = new Thread(runnable, "t3");
        t1.start();
        t2.start();
        t3.start();
        //确保t1.t2.t3正常启动
        Thread.sleep(1000L);

        System.out.println("t1.getState() = " + t1.getState());
        System.out.println("t2.getState() = " + t2.getState());
        System.out.println("t3.getState() = " + t3.getState());

        //调用notifyAll唤醒所有
        Thread t4 = new Thread(() -> {
            synchronized (o) {
                o.notifyAll();
                System.out.println("============调用了notifyAll,代码块还未执行完===============");
                System.out.println("t1.getState() = " + t1.getState());
                System.out.println("t2.getState() = " + t2.getState());
                System.out.println("t3.getState() = " + t3.getState());

            }
        });
        t4.start();
        //确保 调用notifyAll的线程执行完
        Thread.sleep(1000L);
        System.out.println("============调用了notifyAll,代码块已经执行完===============");
        System.out.println("t1.getState() = " + t1.getState());
        System.out.println("t2.getState() = " + t2.getState());
        System.out.println("t3.getState() = " + t3.getState());
    }
}
```

打印结果如下:

``` shell
t1.getState() = WAITING
t2.getState() = WAITING
t3.getState() = WAITING
============调用了notifyAll,代码块还未执行完===============
t1.getState() = BLOCKED
t2.getState() = BLOCKED
t3.getState() = BLOCKED
============调用了notifyAll,代码块已经执行完===============
t1.getState() = BLOCKED
t2.getState() = BLOCKED
t3.getState() = RUNNABLE
```

1. 线程t1,t2,t3先启动,因为调用了wait()方法而wait()方法释放对象锁,进入等待池中等待唤醒,此时三个线程的状态都是```WAITING```状态
2. t4线程调用notifyAll()唤醒了t1,t2,t3线程,在还未执行完所有的同步代码快中打印三个线程的状态都为```BLOCKED```,三个线程都已经进入了锁池,等待该线程把对象锁释放.
3. t4线程执行完毕,退出同步代码块,释放对象锁.线程t1,t2,t3同时抢对象锁,但是谁抢到这个是随机的.结果是t3抢到锁,进入无限while循环,t1,t2因为没有抢到锁,留在锁池等待对象锁.三个线程的状态分别为```BLOCKED```,```BLOCKED```,```RUNNABLE```与上述过程吻合.

## 当前线程调用wait()只会释放当前共享变量上面的锁

当前线程调用共享对象上的wait时,只会释放当前共享变量上面的锁.如果该线程还带有其他共享对象的锁,是不会释放的.如果这点没有注意,很容易发生死锁.

``` java
public class Ch3 {
    public static void main(String[] args) {
        Object resourceA = new Object();
        Object resourceB = new Object();

        Runnable runnable = () -> {
            synchronized (resourceA){
                System.out.println(Thread.currentThread().getName()+":获取resourceA的锁");
                synchronized (resourceB){
                    System.out.println(Thread.currentThread().getName()+":获取resourceB的锁");
                    try {
                        resourceA.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName()+":wait唤醒之后执行内容");
                }
            }
        };

        Thread t1 = new Thread(runnable, "t1");
        Thread t2 = new Thread(runnable, "t2");
        t1.start();
        t2.start();
    }
}
```

上面的代码执行结果如下:

```java
t1:获取resourceA的锁
t1:获取resourceB的锁
t2:获取resourceA的锁
```

代码最后卡死在t2获取```resourceB```上.因为```t1```在执行只释放了```resourceA```对象上的锁,而```resourceB```上面的锁并未被释放,最后导致另一个线程在等待锁的释放产生了死锁.

## 使用案例-生产者和消费者

生产者消费者问题是一个很经典的问题,它就是一个或者多个生产者生产东西,然后一个或者多个消费者去消费东西.需要保证生产者生产的东西不能超过我们设置的最大容量,同时也要保证消费者消费时不能过度消费.同时还有一点非常重要,就是生产者和消费者不能最后同时不生产和不消费,导致程序"卡死".

### 一个生产者和一个消费者

示例代码如下:

```java
package com.buydeem.ch2;

/**
 * 生产者消费者问题 一个生产者一个消费者
 * Created by zengchao on 2020/6/30.
 */
public class App1 {
    public static void main(String[] args) {
        Repository repository = new Repository();
        Thread p1 = new Thread(new Producer(repository, 200));
        p1.setName("生产者1");
        Thread c1 = new Thread(new Consumer(repository, 200));
        c1.setName("消费者1");

        p1.start();
        c1.start();

        while (true) {

        }
    }
}

/**
 * 仓库对象
 */
class Repository {
    /**
     * 当前数量
     */
    private Integer current = 0;
    /**
     * 最大数量
     */
    private static final Integer MAX = 1;

    public Repository() {
    }

    /**
     * 添加
     */
    public synchronized void add() throws InterruptedException {
        if (current >= MAX) {
            System.out.println(Thread.currentThread().getName() + ":仓库满了,等待消费者消费之后再添加");
            wait();
        }
        current = current + 1;
        System.out.println(Thread.currentThread().getName() + ":添加成功,当前仓库物品数量:" + current);
        notify();
    }

    /**
     * 取出
     */
    public synchronized void remove() throws InterruptedException {
        if (current <= 0) {
            System.out.println(Thread.currentThread().getName() + ":仓库为空,等待生产者生产再取出");
            wait();
        }
        current = current - 1;
        System.out.println(Thread.currentThread().getName() + ":取出成功,当前仓库物品数量:" + current);
        notify();
    }
}

/**
 * 生产者
 */
class Producer implements Runnable {
    private Repository repository;
    private Integer count;

    public Producer(Repository repository, Integer count) {
        this.repository = repository;
        this.count = count;
    }

    @Override
    public void run() {
        for (int i = 0; i < count; i++) {
            try {
                repository.add();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

/**
 * 消费者
 */
class Consumer implements Runnable {
    private Repository repository;
    private Integer count;

    public Consumer(Repository repository, Integer count) {
        this.repository = repository;
        this.count = count;
    }

    @Override
    public void run() {
        for (int i = 0; i < count; i++) {
            try {
                repository.remove();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

上面的代码打印结果如下:

```java
生产者1:添加成功,当前仓库物品数量:1
生产者1:仓库满了,等待消费者消费之后再添加
消费者1:取出成功,当前仓库物品数量:0
消费者1:仓库为空,等待生产者生产再取出
生产者1:添加成功,当前仓库物品数量:1
生产者1:仓库满了,等待消费者消费之后再添加
...
```

后面还有很多就不全部贴出来了.而且计算```消费者1:取出成功,当前仓库物品数量```这个的打印次数刚好是200次,而```生产者1:添加成功,当前仓库物品数量```打印次数也是200次.而且仓库的最大数量没有超过2,最小数量也没有小于0.说明这个代码在一个生产者一个消费者的情况下没有问题.那么我们增加生产者或者消费再试试,看是否还能正常运行.

### 一个生产者和多个消费者引发的过度消费问题

修改代码如下:

``` java
    public static void main(String[] args) {
        Repository repository = new Repository();
        Thread p1 = new Thread(new Producer(repository, 200));
        p1.setName("生产者1");
        Thread c1 = new Thread(new Consumer(repository, 100));
        c1.setName("消费者1");
        Thread c2 = new Thread(new Consumer(repository, 100));
        c2.setName("消费者2");

        p1.start();
        c1.start();
        c2.start();

        while (true) {

        }
    }
```

与之前代码不同的时,添加一个消费者线程,再次执行.截取部分异常结果如下:

```java
消费者2:取出成功,当前仓库物品数量:0
消费者2:仓库为空,等待生产者生产再取出
消费者1:取出成功,当前仓库物品数量:-1
消费者1:仓库为空,等待生产者生产再取出
消费者2:取出成功,当前仓库物品数量:-2
消费者2:仓库为空,等待生产者生产再取出
消费者1:取出成功,当前仓库物品数量:-3
```

上面的结果出现了明显的消费情况,这是什么原因导致的呢?  

1. 假设c1现在是```WAITING(即调用了wait状态)```,p1同c1一样现在是```WAITING```状态,而c2处于```RUNNABLE```中.  
2. c2执行remove减库存,将库存减到了0然后调用了notify.
3. 此时唤醒谁这个我们没法控制,但是我们能知道的是被唤醒的肯定在p1和c1中.我们假设唤醒的是c1,且c1在和c2在竞争锁时,c1获取到了对象锁(这里的锁就是repository对象).
4. c1被唤醒,因为之前是在```wait()```处进入```WAITING```状态的,被唤醒了且获取到了锁,这个时候继续执行```wait()```后面的代码逻辑,减库存(此时库存已经为0了),打印当前的库存就为-1了.

从上面的分析过程可以看出,这里面的问题主要在于判断库存使用```if```导致的.当我们被唤醒后,不应该直接执行后面的业务逻辑,而应该重新判断条件.因为被唤醒后,判断条件的值可能已经发生了变化.如果想避免这种情况发生,我们修改生成者和消费者中的代码如下:

```java
    /**
     * 添加
     */
    public synchronized void add() throws InterruptedException {
        while (current >= MAX) {
            System.out.println(Thread.currentThread().getName() + ":仓库满了,等待消费者消费之后再添加");
            wait();
        }
        current = current + 1;
        System.out.println(Thread.currentThread().getName() + ":添加成功,当前仓库物品数量:" + current);
        notify();
    }

    /**
     * 取出
     */
    public synchronized void remove() throws InterruptedException {
        while (current <= 0) {
            System.out.println(Thread.currentThread().getName() + ":仓库为空,等待生产者生产再取出");
            wait();
        }
        current = current - 1;
        System.out.println(Thread.currentThread().getName() + ":取出成功,当前仓库物品数量:" + current);
        notify();
    }
```

我们只是简单的将```if```修改成```while```,这样醒来后再重新判断条件,就可以避免过度消费或者过度生产的情况.

### 一个生产者和多个消费者引发的假死

上面的代码修改完之后,再次验证是否还会过度消费.再次执行代码,当控制台不再打印时,拷贝打印结果统计,发现生产的条数和消费的条数并没有达到预期的200条.多运行几次,发现每次的生产和消费条数都不足200条.这是为什么呢?  
使用jvisualvm工具查询线程状态,如下图所示:  
![生产者和消费者线程状态](https://i.loli.net/2020/06/30/jnWik4MYPf7pwe5.png)  
从上图可以看出生产者线程和消费者线程都进入了```WAITING```状态,导致程序"假死".下面来分析这种情况是如何发生的.  

1. 假设c1现在是```WAITING(即调用了wait状态)```,p1同c1一样现在是```WAITING```状态,而c2处于```RUNNABLE```中.  
2. c2执行remove减库存,将库存减到了0然后调用了notify.
3. 此时唤醒谁这个我们没法控制,但是我们能知道的是被唤醒的肯定在p1和c1中.我们假设唤醒的是c1.这个时候p1,c1,c2的状态分别为```WAITING```,```BLOCK```,```BLOCK```.这个时候不管是c1还是c2获取到锁都一样.我们假设c1获取到了锁.
4. c1获取到锁,因为库存为0,所以调用wait()进入```WAITING```,然后c2获取到锁继续执行.
5. c2获取到锁,同样因为库存不足为0调用wait()进入```WAITING```.这个时候p1,c1,c2的状态分别为```WAITING```,```BLOCK```,```BLOCK```.

通过上面的分析我们知道了为什么会导致生产者和消费者全部进入```WAITING```,主要就是因为我们没办法控制notify唤醒的是生产者还是消费者.如果生产者全部是```WAITING```状态,而消费者notify随机唤醒的不是生产者,那么就有可能导致最后的消费者也全部进入```WAITING```状态.当做所有的生产者和消费者都为```WAITING```时,程序进入了"假死状态".  
既然知道了为何产生,那么我们就很好解决了.java中还提供了一个方法用来唤醒所有等待在同一对象锁上的线程notifyAll().通过这个方法,我们可以将生产者消费者一同唤醒,就会不出现"假死"状况了.  

```java
/**
     * 添加
     */
    public synchronized void add() throws InterruptedException {
        //修改if => while
        while (current >= MAX) {
            System.out.println(Thread.currentThread().getName() + ":仓库满了,等待消费者消费之后再添加");
            wait();
        }
        current = current + 1;
        System.out.println(Thread.currentThread().getName() + ":添加成功,当前仓库物品数量:" + current);
        //修改notify=>notifyAll
        notifyAll();
    }

    /**
     * 取出
     */
    public synchronized void remove() throws InterruptedException {
        //修改if => while
        while (current <= 0) {
            System.out.println(Thread.currentThread().getName() + ":仓库为空,等待生产者生产再取出");
            wait();
        }
        current = current - 1;
        System.out.println(Thread.currentThread().getName() + ":取出成功,当前仓库物品数量:" + current);
        //修改notify=>notifyAll
        notifyAll();
    }
```

再次运行上面的代码,添加多个消费者和生产者运行,没有发生异常数据,同时也不会进入"假死".

### 总结分析

上面简单的实现了生产者和消费者模型.需要注意的点主要就是两个:  

- ```wait()```之后代码会继续执行后续逻辑.而判断条件是在进入```WAITING```之前就判断了,很可能再次执行时,判断逻辑已经不再成立.所以推荐使用```while```代替```if```判断.
- ```notify()```在多个生产者或者多个消费者的情况下请使用```notifyAll()```,它能避免所有线程全部进入```WAITING```的情况.
