# ReentrantLock

## 简叙

我们可以使用```synchronized```来完成线程间的同步保证共享变量的线程安全,而使用```synchronized```加锁和解锁都是通过隐式的,简单的说就是我们不需要手动的获取锁和释放锁.而我们现在锁的```ReentrantLock```也能实现```synchronized```类似的功能,同时还提供更强大的功能.  
关于```synchronized```可以参照我之前写的笔记:[synchronized关键字](https://www.jianshu.com/p/bb70716facb5)和[锁优化](https://www.jianshu.com/p/2511b433aea2).  
```ReentrantLock```内部是通过```AQS```实现的,但是本文暂时不讲内部实现,只讲如何使用和使用的注意事项.后面将会有文章专门介绍```AQS```.

## 基本特性

- ```互斥性```:即同一时刻只能有一个线程能进入同步的代码块,其他所有想进入同步代码块中的线程必须等到占有锁的线程离开才可以抢到锁进入.
- ```可重入性```:即一个线程如果已经获取到锁,它可以再次进入同一锁控制的代码块.

上面的特性我们使用一个简单的例子来说明:

```java
public class App1 {
    public static void main(String[] args) {
        //创建锁
        ReentrantLock lock = new ReentrantLock();

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                //第一次抢锁
                System.out.println(Thread.currentThread().getName()+":第一次抢锁");
                lock.lock();
                System.out.println(Thread.currentThread().getName()+":第一次获取到锁");

                //再次抢锁
                System.out.println(Thread.currentThread().getName()+":第二次抢锁");
                lock.lock();
                System.out.println(Thread.currentThread().getName()+":第二次获取到锁");
            }
        };

        Thread t1 = new Thread(runnable);
        t1.setName("t1");
        t1.start();

        Thread t2 = new Thread(runnable);
        t2.setName("t2");
        t2.start();
    }
}
```

打印结果如下所示:

```java
t1:第一次抢锁
t2:第一次抢锁
t1:第一次获取到锁
t1:第二次抢锁
t1:第二次获取到锁
```

线程```t1```和线程```t2```调用```lock```去获取锁,但是线程```t1```获取到了```t2```没有获取到.因为```互斥性```所以```t2```只能阻塞等待.```t1```再次调用```lock```获取锁,因为```可重入性```所以```t1```可以再次获取到锁.而```t1```执行完成后并没有释放锁,所以线程```t2```将会一直阻塞等待下去.

## 常用方法

### 获取锁

- ```lock```:获取锁.如果该锁没有被其他线程占有,将直接获取到锁并返回.如果锁已被其他线程占有,那么该线程将进入阻塞等待直到锁被释放.
- ```lockInterruptibly```:与```lock```不同的点在于它会响应中断.如果当前线程已经被中断或者在等待的过程中被中断了,将抛出```InterruptedException```异常,同时还会清除线程的中断状态.
- ```tryLock```:尝试获取锁.如果当前锁没有被其他线程占有则立马返回true并获取锁.如果已被占有将返回false.该方法不会导致调用线程阻塞等待.
- ```tryLock(long timeout,TimeUnit unit)```:在指定超时时间内尝试获取锁.与```tryLock```类似,如果超时时间到了还未获取到锁将结束等待并返回fasle.在等待过程中线程会响应线程中断信号,并抛出```InterruptedException```异常,同时还会清除线程的中断状态.

```java
public class App2 {
    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //调用lock获取锁
                lock.lock();
                System.out.println(Thread.currentThread().getName()+":获取到锁");
            }
        });
        t1.setName("t1");
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //调用lockInterruptibly获取锁
                    lock.lockInterruptibly();
                    System.out.println(Thread.currentThread().getName()+":获取到锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    System.out.println(Thread.currentThread().getName()+":中断退出");
                }
            }
        });
        t2.setName("t2");

        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //调用tryLock获取锁
                    boolean success = lock.tryLock(1, TimeUnit.SECONDS);
                    if (success){
                        System.out.println(Thread.currentThread().getName()+":获取到锁");
                    }else {
                        System.out.println(Thread.currentThread().getName()+":获取到锁超时");
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t3.setName("t3");

        t1.start();
        //保证线程1先获取到锁
        Thread.sleep(500);
        t2.start();
        t3.start();

        //中断线程t2
        Thread.sleep(500);
        System.out.println(Thread.currentThread().getName()+":中断线程t2");
        t2.interrupt();
    }
}
```

最后打印结果如下:

```java
t1:获取到锁
main:中断线程t2
t2:中断退出
java.lang.InterruptedException
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
    at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
    at com.buydeem.lock.App2$2.run(App2.java:25)
    at java.lang.Thread.run(Thread.java:745)
t3:获取锁超时
```

上面示例代码中,线程t1通过调用```lock```方法获取锁成功,然后t1执行完后面逻辑并没有释放锁.t2调用```lockInterruptibly```获取锁,后面以为主线程调用```t2.interrupt()```中断线程t2,导致线程t2抛出中断异常而退出等待.线程t3调用```tryLock```获取锁,在超时时间到了之后没能获取到锁然后退出等待.

### 释放锁

释放锁通过调用```unlock```.因为```ReentrantLock```为可重入锁,获取锁的次数与释放锁必须相等,否则将导致其他线程永远也无法释放锁.同时调用```unlock```方法必须先获取到锁,如果没有获取到锁调用将抛出```IllegalMonitorStateException```.

```java
public class App3 {
    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //第一次获取锁
                lock.lock();
                System.out.println(Thread.currentThread().getName() + ":第一次获取锁");
                //第二次获取锁
                lock.lock();
                System.out.println(Thread.currentThread().getName() + ":第二次获取锁");
                //第一次释放锁
                lock.unlock();
                System.out.println(Thread.currentThread().getName() + ":第一次释放锁");
            }
        });
        t1.setName("t1");
        t1.start();

        //保证t1先获取到锁
        Thread.sleep(500);

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();
                System.out.println(Thread.currentThread().getName() + ":获取锁");
            }
        });
        t2.setName("t2");
        t2.start();

        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.unlock();
            }
        });
        t3.setName("t3");
        t3.start();
    }
}
```

打印结果如下:

```java
t1:第一次获取锁
t1:第二次获取锁
t1:第一次释放锁
Exception in thread "t3" java.lang.IllegalMonitorStateException
    at java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:151)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1261)
    at java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:457)
    at com.buydeem.lock.App3$3.run(App3.java:45)
    at java.lang.Thread.run(Thread.java:745)
```

线程t1获取了两次锁,但是只调用了一次```unlock```导致锁没有被释放.所以线程t2无法获取到锁,程序也无法结束.线程t3在没有获取到锁的情况下调用```unlock```导致抛出```IllegalMonitorStateException```异常.

## condition

我们之前说过```ReentrantLock```可以提供```synchronized```相同的功能,甚至提供更强大的功能.而```condition```就是其中之一.在之前的文章中我介绍过[wait和notify](https://www.jianshu.com/p/f37b9e0a3499),通过该方式可以实现线程间的通信.而我们现在要说的```condition```也同样提供一样的功能,而且它提供的功能更加强大.如果懂```wait和notify```理解起来将会特别简单,如果不懂的话看完```condition```再理解```wait和notify```也会很简单.  
```condition```主要提供了两种API.一种让当前线程放弃所等待唤醒,另一种则是唤醒其他等待功能.这些API就是```await()```和```signal()```,类比于```synchronized```中的```wait()```和```notify()```.正是这种睡眠唤醒机制所以我们可以用它来实现线程间的协调通信.我们先简单的使用一个示例来展示如何实现线程间的协调通信.下面的例子使用两个线程交替打印数字,代码如下所示:

```java
//使用wait和notify机制实现
public class App5 {
    //记录当前共享变量的值
    private static Integer count = 0;
    public static void main(String[] args) {
        //创建锁对象
        Object o = new Object();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                while (true) {
                    //同步代码块
                    synchronized (o) {
                        try {
                            //休眠1s,避免打印速度太快
                            Thread.sleep(1000);
                            count = count + 1;
                            //打印当前线程的名字和共享变量的值
                            System.out.println(Thread.currentThread().getName() + ":" + count);
                            //唤醒等待在o对象上的锁
                            o.notify();
                            //当前线程等待
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        };

        //创建两个线程
        Thread t1 = new Thread(runnable);
        t1.setName("t1");
        Thread t2 = new Thread(runnable);
        t2.setName("t2");
        //启动线程
        t1.start();
        t2.start();

    }
}
```

上面的代码我们是通过```wait和notify```来实现的,我们下面通过```condition```的方式来实现,代码如下:  

```java
//使用condition方式
public class App4 {
    //记录当前共享变量的值
    private static Integer count = 0;
    public static void main(String[] args) {
        //创建锁
        ReentrantLock lock = new ReentrantLock();
        //创建condition
        Condition condition = lock.newCondition();
        //创建任务
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                while (true){
                    try {
                        //休眠1s,避免打印速度太快
                        Thread.sleep(1000);
                        //获取锁
                        lock.lock();
                        count = count + 1;
                        //打印当前线程名称和共享变量count的值
                        System.out.println(Thread.currentThread().getName()+":"+count);
                        //唤醒在condition上等待的线程
                        condition.signal();
                        //当前线程等待
                        condition.await();
                        //手动释放锁
                        lock.unlock();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        //创建两个线程
        Thread t1 = new Thread(runnable);
        t1.setName("t1");
        Thread t2 = new Thread(runnable);
        t2.setName("t2");
        //启动线程
        t1.start();
        t2.start();
    }
}
```

上面的两种方式都能实现两个线程交替打印数字的功能.而我们在使用```synchronized```有些限制和注意点在使用```condition```的时候同样需要注意.在调用```await```方法前```必须先获取锁才能调用await```

```java
public class App6 {
    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        condition.await();
    }
}
```

上面的代码运行会抛出```IllegalMonitorStateException```异常.该异常的原因就是我们在调用```await```方法时我们并未获取到锁,而且调用```await```是会释放对象锁的.为什么要释放锁呢?因为如果你调用```await```不释放获取的锁,那么你唤醒其他线程但是它又不能获取到锁还是会阻塞等待获取锁,那```await```就没有存在的意义了.我们用下面这段代码来证明调用```await```是会成功释放锁的.代码如下:  

```java
public class App7 {
    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        //创建线程1
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //获取锁
                lock.lock();
                System.out.println(Thread.currentThread().getName()+":成功获取到锁");
                //调用await()等待,并且会释放锁
                try {
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t1.setName("t1");
        t1.start();
        //确保线程1先获取到锁
        Thread.sleep(500);

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                //获取锁
                lock.lock();
                System.out.println(Thread.currentThread().getName()+":成功获取到锁");
                //释放锁
                lock.unlock();
                System.out.println(Thread.currentThread().getName()+":成功释放掉锁");
            }
        });
        t2.setName("t2");
        t2.start();
    }
}
```

上面的代码主要内容为:线程```t1```先获取到了锁,然后调用```await```释放手中的锁.而线程```t2```获取锁,然后打印信息.最后的打印结果如下所示:  

```java
t1:成功获取到锁
t2:成功获取到锁
t2:成功释放掉锁
```

从打印结果可以知道```t1```获取了锁,调用了```await```方法释放掉了锁.如果```await```方法不会释放锁,那么```t2```线程永远就拿不到锁了,更加会不打印出下面的信息.这点性质跟调用```wait```一样也是需要先获取锁才能调用.  

如果看完上面的内容你会发现```condition```并没有比```synchronized```强大,现在我们就来介绍它比```synchronized```强大的点:```可以指定唤醒的线程```.  
使用```notify```或者```notifyAll```我们可以唤醒线程,但是我们无法指定唤醒线程.在经典的生产者消费者模型中,我们无法确定唤醒的线程是生产者还是消费者.在我以前写的文章中使用```wait```和```notify```实现过生产者和消费者模型,请参考[这篇](https://www.jianshu.com/p/f37b9e0a3499)文章.现在我们使用```condition```这种方式重新来实现该模型.

```java
package com.buydeem.lock;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Created by zengchao on 2020/7/16.
 */
public class App8 {
    public static void main(String[] args) {
        //创建仓库
        Repository repository = new Repository();
        //创建三个生产者,每个生产10000
        Thread p1 = new Thread(new Producer(repository, 10000));
        p1.setName("p1");
        Thread p2 = new Thread(new Producer(repository, 10000));
        p2.setName("p2");
        Thread p3 = new Thread(new Producer(repository, 10000));
        p3.setName("p3");
        // 创建2个消费,每个消费15000
        Thread c1 = new Thread(new Consumer(repository,15000));
        c1.setName("c1");
        Thread c2 = new Thread(new Consumer(repository,15000));
        c2.setName("c2");
        //启动线程
        p1.start();
        p2.start();
        c1.start();
        c2.start();
    }
}

/**
 * 仓库对象
 */
class Repository {
    /**
     * 当前仓库产品数量
     */
    private Integer current = 0;
    /**
     * 仓库最大容量
     */
    private static final Integer MAX = 10;

    private ReentrantLock lock = new ReentrantLock();
    private Condition productCondition = lock.newCondition();
    private Condition consumerCondition = lock.newCondition();

    /**
     * 添加产品
     */
    public void add() throws InterruptedException {
        //获取锁
        lock.lock();
        //判断仓库是否满了
        while (current >= MAX){
            System.out.println(String.format("%s:当前仓库已满,等消费者消费之后才能添加",Thread.currentThread().getName()));
            //仓库满了进入等待
            productCondition.await();
        }
        //当前仓库产品数量+1
        current = current + 1;
        System.out.println(String.format("%s:生产一个产品,当前数量:[%d]",Thread.currentThread().getName(),current));
        //唤醒消费者消费
        consumerCondition.signal();
        //释放锁
        lock.unlock();
    }

    /**
     * 消费产品
     */
    public void remove() throws InterruptedException {
        //获取锁
        lock.lock();
        //判断仓库是否有产品
        while (current <= 0){
            //仓库没有产品可以消费
            System.out.println(String.format("%s:当前仓库已空,等待生产者生产之后才能消费",Thread.currentThread().getName()));
            //仓库空了进入等待
            consumerCondition.await();
        }
        //当前仓库数量-1
        current = current - 1;
        System.out.println(String.format("%s:消费一个产品,当前数量:[%d]",Thread.currentThread().getName(),current));
        //唤醒生产者生产
        productCondition.signal();
        //解锁
        lock.unlock();
    }
}

/**
 * 生产者
 */
class Producer implements Runnable{
    /**
     * 仓库
     */
    private Repository repository;
    /**
     * 生产次数
     */
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
class Consumer implements Runnable{
    /**
     * 仓库
     */
    private Repository repository;
    /**
     * 消费次数
     */
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

上面的例子中,我们创建两个```condition```对象```productCondition```和```consumerCondition```.通过```condition```来唤醒线程和让线程等待.我们可以明确的让生产者或者消费者唤醒或者线程.而使用```synchronized```通过```wait```和```notify```是无法指定是生产者还是消费者.这就是```condition```的强大和灵活之处.
