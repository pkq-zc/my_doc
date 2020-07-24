# AQS(上)-独占模式获取锁

## 简叙

之前我写过很多关于JUC下各种锁的使用文章,但是都没说是如何实现的.如果你去看```ReentrantLock```的源码,你会发现它内部有一个```Sync```类,它继承了```AbstractQueuedSynchronizer```这个抽象类实现了该类的某些方法.这个就是我们今天要说的```AQS```.我们一定先要搞懂这个类才能真正的了解到```ReentrantLock```它是如何实现的.  

## 阅读预备知识点

- ```CAS```:需要知道什么是CAS.简单的说就是```比较```和```交换```.举个简单的例子:例如你去更新数据中的订单状态为```未支付```的订单为```已支付```.你的sql语句必须是```update t_order set status = '已支付' where order_id = 'xxxx' and status = '未支付'```.而不应该是```update t_order set status = '已支付' where order_id = 'xxxx'```.如果还是不理解可以查看我们之前写的关于[CAS](https://www.jianshu.com/p/171652d2a971)的文章.

- ```volatile```:需要知道```volatile```关键字的作用.简单的说使用它修饰的变量,只要值发生改变在其他线程能立马看见改变后的值.可以参考我之前写的关于[volatile](https://www.jianshu.com/p/a11e2c6a89aa)相关的文章.

- ```CLH队列```:一种基于单向```链表```实现的队列.基于该队列可以实现一个简单高效的自旋锁.在阅读源码中我们同时可以思考AQS是如何来实现可重入的.关于CLH队列的介绍可以参照我之前写的文章[CLHLOCK实现](https://www.jianshu.com/p/36518c913995)

- ```LockSupport```:用来创建锁和其他同步类的线程阻塞原语.简单的说就是用来阻塞和唤醒线程的.具体介绍可以查看[pack和unpack](https://www.jianshu.com/p/d643be130220).

上面说的预备知识请保证先经历弄清楚,如果不是很清楚不太建议看关于```AQS```的实现.因为如果上面的知识点你不清楚,你看完也不知道为什么会这样,到处都是各种疑问.当然还有其他知识点我可能没写上,但是这几个比较重要的还请先弄清楚或者说至少知道他们的作用.

## 解析

**注意**:本文是基于JDK1.8所写,因为我没有看到其他版本的JDK源码,如果你使用的不是JDK1.8可能会稍有不同,当并不影响我们了解```AQS```的实现.

### 为什么AQS是abstract类

```AbstractQueuedSynchronizer```被声明成一个```abstract```类,而在java中```abstract```一般代表这个类有部分方法未被实现,我们在用的时候需要根据自己的具体需求来实现.但是你查看源码可以发现并没有方法被声明成```abstract```.但是有部分方法内部实现为:```throw new UnsupportedOperationException();```.这部分方法主要如下:  

```java
//独占方式获取资源.true表示成功,false表示失败
protected boolean tryAcquire(int arg)
//独占方式释放资源.true表示成功,false表示失败
protected boolean tryRelease(int arg)
//共享方式获取资源.返回值大于等于0是代表成功,小于0代表失败
protected int tryAcquireShared(int arg)
//共享方式释放锁.true表示成,false表示失败
protected boolean tryReleaseShared(int arg)
//对于调用的线程同步是以独占的形式进行的返回true,否则返回false.
protected boolean isHeldExclusively()
```

上面这些方法主要可以分为```SHARED(共享)```和```EXCLUSIVE(独占)```这两大类.而我们自定义的同步器一般就是独占式例如```ReentrantLock```或者共享式```CountDownLatch```这两大类.我们只需要实现```tryAcquire-tryRelease```或者```tryAcquireShared-tryReleaseShared```即可.但是也有同时实现这两种的例如```ReentrantReadWriteLock```.

### CLH实现

```AQS```中的CLH队列是CLH的一种变种实现.在我之前写的```CLHLOCK```文章中指出了```CLHLOCK```是无法实现锁的可重入的.而```AQS```是支持锁重入的.下面介绍```AQS```中的```CLH```是如何实现的.

#### Node

```AQS```内部有一个```Node```静态内部类,该类就是组成```CLH```队列的节点.```它是对每一个等待获取共享资源的线程的封装```.先看下面的源码我们注释来了解一个```Node```具体封装了哪些内容.

```java
    static final class Node {
        //共享模式的标记
        static final Node SHARED = new Node();
        //独占模式的标记
        static final Node EXCLUSIVE = null;
        //表示当前节点已经被取消了
        static final int CANCELLED =  1;
        //表示当前节点的后继节点等待唤醒.后继节点在进入等待前会将它的前节点(前节点有效)设置为该状态
        static final int SIGNAL    = -1;
        //表示该节点等待在condition上,当其他线程调用signal()方法后,condition状态的节点将从等待队列转移到同步队列.
        static final int CONDITION = -2;
        //共享模式下,当前节点唤醒后继节点及其后继节点的后继节点
        static final int PROPAGATE = -3;
        //节点的等待状态.默认为0.
        volatile int waitStatus;
        //前节点
        volatile Node prev;
        //后继节点
        volatile Node next;
        //线程
        volatile Thread thread;
        //链接在条件队列等待的下一个节点或者是特殊值SHARD
        Node nextWaiter;
        // 判断当前节点是否是共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        //返回前驱节点
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
    }
```

从上面的Node的结构可以看出Node组成的队列是一个基于```双向链表```实现的队列.每个因为获取共享资源的线程被阻塞等待时都将被封装成一个```Node```添加到等待队列.线程本身以及线程的等待状态都被封装在该队列中.

#### state

上面说了封装等待线程信息的Node,但是原生的```CLH```实现的锁没法实现```重入性```的.但是```AQS```中却实现了```重入性```,```AQS```是如何实现可重入性的呢?同时我们所说的共享资源到底是什么呢?  

上面两个问题的答案就是:```state```.在```AQS```内部有一个变量,它的定义如下:  

```java
private volatile int state;
```

```AQS```就是使用```state```来表示共享资源.而且需要注意的是它是使用```volatile```来修饰的.其实不仅仅只有这个变量是使用```volatile```来修饰,还有很多变量都是使用了该词修饰,主要作用还是为了多线程修改值在其他线程中能感知到.同时```AQS```还提供了一个重要的方法用来修改```state```的值,具体方法如下:

```java
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

上面是一个```CAS```操作,```expect```预期值也可以理解为旧值,```update```要更新的值.在更新```state```值的时候会先比较目前```state```的值是不是和```expect```值一样,如果一样说明没有其他线程修改过该值则更新成功.如果不一样则说明有其他线程修改过```state```的值则该次修改失败并返回```false```.在```AQS```中海油其他地方也使用```CAS```操作,后面遇见了不会再做解释了.  

上面说明```state```用来表示共享资源,还有另外一个问题就是为什么能重入?我摘取```ReentrantLock```中一段源码来解释,该段代码为内部类```Sync```中非公平模式获取资源的实现.

```java
// acquires 代表需要获取的资源数
final boolean nonfairTryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取state的值
            int c = getState();
            //如果state=0,说明资源没有被任何线程占用.如果大于0,说明已经有线程获取了资源.需要说明是state代表资源,但是具体意思会因为共享模式和独占模式而有所不同
            if (c == 0) {
                //CAS操作更新资源值
                if (compareAndSetState(0, acquires)) {
                    //将当前线程设置成当共享资源的拥有者.其实就是将当前线程保存到AQS实例中
                    setExclusiveOwnerThread(current);
                    //返回获取资源成功
                    return true;
                }
            }
            //说明已经有线程占用了共享资源,且是同一线程
            else if (current == getExclusiveOwnerThread()) {
                //重入重入重入重入重入重入重入
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                //修改state的值,很巧妙的实现了重入
                setState(nextc);
                return true;
            }
            //占用资源的不是当前线程,说明该次获取锁失败
            return false;
        }
```

上面代码解释了如何```重入性```是如何实现的.这里面有两个关键点就在于```state```和```ownerThread```.当线程再次重入获取锁时,我们只需要比较当前获取锁的线程是不是```AQS```的拥有者时就能判断是不是```重入```.如果是只需要更新```state```的值即可,这个设计真的牛逼.而如何保存```AQS```的拥有者时,这个实现也很简单,代码如下:

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
    private static final long serialVersionUID = 3737899427754241961L;
    protected AbstractOwnableSynchronizer() { }
    private transient Thread exclusiveOwnerThread;
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

```AbstractQueuedSynchronizer```继承了这个类,这个类就是通过一个属性用来保存当前拥有线程的.  

#### 结构图

通过上面的图心里大概应该有了模糊的概念了.下面这张图将展示```AQS```内部的等待队列的结构:  

![CLH变种队列](https://wx2.sbimg.cn/2020/07/22/DJ7LO.png)  

```AQS```内部使用```head```和```tail```用来保存队列的头结点和尾节点.使用```state```代表资源.每个```Node```用来保存等待获取共享资源的线程.这就是```AQS```的核心内容了.

## 代码解析

从上面的内容我们知道了获取资源有两种模式,一种为独占模式,另一种为共享模式.我们下面就根据模式的不同做详细的代码分析整个流程:

### 独占模式

#### 独占模式获取锁

如果我们要分析源码首先要找到一个入口,这样分析起来就比较容易了.独占模式获取共享资源的入口一共有下面三个:

- ```acquire(int arg)```:这个方法是不响应中断和超时的.
- ```acquireInterruptibly(int arg)```:这个是响应中断的.
- ```tryAcquireNanos(int arg, long nanosTimeout)```:这个是响应超时的.

上面三个方法就是获取独占模式下获取共享资源的入口了,具体实现有部分差别,但是核心过程都差不多.我这里只分析```acquire```方法,其他的如果你自己有兴趣可以自己分析.

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //如果线程被中断过则调用 Thread.currentThread().interrupt()
            selfInterrupt();
    }
```

1. 首先线程尝试去获取锁,如果获取成功则继续执行业务逻辑,失败则进行下一步.
2. 尝试抢锁失败,调用```addWaiter()```将线程添加到尾部并标记自己为独占模式
3. acquireQueued线程阻塞在等待队列.

```java
    private Node addWaiter(Node mode) {
        //创建Node节点,根据mode参数设置该节点是共享还是排他模式
        Node node = new Node(Thread.currentThread(), mode);
        //获取到尾节点
        Node pred = tail;
        if (pred != null) {
            //尾节点不为空,说明当前队列不为空
            //设置当前节点的前节点为获取到的尾节点
            node.prev = pred;
            //CAS操作将当前节点设置成尾节点
            if (compareAndSetTail(pred, node)) {
                //设置成功之后将前置节点的尾节点设置成自己
                pred.next = node;
                //返回当前线程封装的Node实例
                return node;
            }
        }
        //可能尾节点为空或者设置当前节点为尾节点失败
        enq(node);
        return node;
    }

    //快速插入队列失败,使用自旋方式再试
    private Node enq(final Node node) {
        //for循环自旋
        for (;;) {
            //获取当前队列的尾节点
            Node t = tail;
            if (t == null) {
                //尾节点为空,创建一个新节点作为头尾节点.CAS操作.然后再次循环
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //已经存在尾节点了,将等待线程的node的前置节点为尾节点
                node.prev = t;
                //CAS操作将自己设置成尾节点,失败则自旋重试
                if (compareAndSetTail(t, node)) {
                    //将自己设置成为节点成功,将前置节点的尾节点设置成自己,退出自旋
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

上面代码展示了将当前线程封装成```Node```实例,并将它添加到队列尾部的操作.在这个过程中首先会在```addWaiter```快速的插入一次,如果这次失败则进入```enq```自旋插入.这里面的精髓就在于```自旋```和```CAS```.  
完成节点的插入之后,下面要进行的就是使线程在队列中阻塞等待.阻塞等待知道其他线程彻底释放了资源然后唤醒自己.  

```java
    final boolean acquireQueued(final Node node, int arg) {
        //是否发生了异常
        boolean failed = true;
        try {
            //标记线程是否中断
            boolean interrupted = false;
            //开始自旋
            for (;;) {
                //获取当前节点的前置节点
                final Node p = node.predecessor();
                //如果前置节点是头节点,说明自己实际上是队列的头节点(头节点实际上是个虚节点),那么可以尝试再获取一次锁,如果成功那就不进入等待了
                if (p == head && tryAcquire(arg)) {
                    //获取锁成功了,将头结点设计成自己
                    setHead(node);
                    // 去掉前置节点与当前节点的引用,便于GC回收无用节点
                    p.next = null;
                    //设置为false
                    failed = false;
                    return interrupted;
                }
                //如果自己不是老二或者获取锁失败,那就老老实实等待.但是进入等待前需要告诉前面一个人记得叫醒自己,同时前面没用人需要把他从队列中移除.
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //进入到该代码块说明线程被中断了,修改中断标志
                    interrupted = true;
            }
        } finally {
            //如果获取锁的过程中出现异常,则会取消当前节点
            if (failed)
                cancelAcquire(node);
        }
    }

    //将前置节点的状态设置为-1(SIGNAL)
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取当前置节点的状态
        int ws = pred.waitStatus;
        //前置节点已经为-1了,说明前置节点会叫我们醒来
        if (ws == Node.SIGNAL)
            return true;
        //waitStatus说明节点已经被取消了,需要将无效节点删除掉
        if (ws > 0) {
            //删除当前节点前的无效节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //前置节点正常,将前置节点状态设置成-1,CAS操作
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    //让当前线程进入等待,醒来后返回当前线程是否被中断过
    private final boolean parkAndCheckInterrupt() {
        //调用park使线程进入等待.它醒来的条件为其他线程调用unpack或者线程中断
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

上面代码就是让线程进入等待的全过程.它核心两个操作就是```告诉前置有效节点唤醒自己```和```进入等待```.  

1. 判断当前节点的前置节点是不是头节点,如果当前节点的前置节点是头节点则尝试获取一次锁.
2. 如果步骤1获取锁成功则返回退出了.如果没有获取成功,则从第一步从新再来.
3. 如果前置节点不是头节点,则进入```shouldParkAfterFailedAcquire```逻辑.

```shouldParkAfterFailedAcquire```主要作用就是让自己"安心的休息".怎样才能安心的休息呢?就是将当前节点的前置节点状态设置为```SIGNAL(-1)```.  

1. 获取前置节点的状态,可能为```-1```,```大于0```,```小于或者等于0```这三种情况.
2. ```-1```:不需要再做别的事了,直接返回```true```
3. ```大于0```:只要节点状态大于0说明节点是个无效节点,这个时候会删除当前节点的无效节点.这个里面的逻辑就是一个简单的链表删除逻辑.
4. ```小于或者等于0```:节点都是有效的节点.我们使用CAS的方式将其设置成```SIGNAL```状态即可.

```parkAndCheckInterrupt```让线程进入等待.这个里面代码很简短,它就是通过调用```LockSupport.park(this)```进入等待的.```再次强调文章前的预备知识请一定先了解!!!!```.  

在```finally```代码块中还有一个逻辑就是用来处理获取资源失败的情况.有些博文说这个地方会在线程中断和线程等待超时时就会调用```cancelAcquire```用来取消节点在队列中等待,这个不是```不对```的.这个只会在获取锁的过程中出现异常才会取消当前节点.因为只有线程只有获取到锁和异常才会退出自旋,```acquireQueued```是不会响应超时和中断的.而响应中断和超时的方法为:```doAcquireInterruptibly```和```doAcquireNanos```.

```java
    //取消在队列中等待
    private void cancelAcquire(Node node) {
        //如果节点不存在,则直接忽略
        if (node == null)
            return;
        //将节点上的线程引用设置为null
        node.thread = null;
        //跳过节点前面取消的节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        //获取筛选后的前置节点的后置节点
        Node predNext = pred.next;
        //将当前节点设置成取消状态
        node.waitStatus = Node.CANCELLED;
        //如果当前节点为尾节点,将从后往前的第一个节点设置为尾节点
        if (node == tail && compareAndSetTail(node, pred)) {
            //去掉前置节点对当前节点的引用.
            compareAndSetNext(pred, predNext, null);
        } else {
            //前一个有效节点不是头节点
            int ws;
            //如果当前节点不是老二,1-判断当前节点的前置节点是否为SIGNAL,2-如果不是则将它设置成SIGNAL
            //如果1和2任意一个成功,再判断当前节点是不是null
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                //将当前节点的前置节点的后节点设置为当前节点的后置节点
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //如果当前节点是head的后继节点,或者上述条件不满足,那就唤醒当前节点的后继节点.(这个方法将会在解锁的时候细说)
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

上面便是取消节点在队列中等待的源码.首先看前节点状态是不是无效节点(即```CANCEL```状态),如果是就一直往前遍历直到找到是有效的节点,然后将找到的节点和当前节点关联,接着将当前节点设置成```CANCEL```.接着判断当前节点的位置做不同的处理方式:  

- ```当前节点是尾节点```:直接将前节点设置成尾节点,并把前节的后置节点设置成```null```.
![current node为tail节点.png](https://i.loli.net/2020/07/23/GPnUMYt3dXw5bgf.png)
- ```当前节点是老二```:将当前节点的后置节点设置成自己.
![current node为老二.png](https://i.loli.net/2020/07/23/WY5XuVr2sh7UGdZ.png)
- ```不上上面两种```:将前节点的尾节点设置成当前节点的后节点,将自己的后节点设置成自己.
![current node既不是tail也不是老二.png](https://i.loli.net/2020/07/23/AmfLMHJQOS5IlTK.png)

上面操作中我们只对```next```做了操作并没有对节点的```pred```做修改.因为如果修改```pred```可能导致```pred```指向一个已经被移除的节点.在```shouldParkAfterFailedAcquire```方法中我们可能会做下面这个操作:  

```java
    if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        }
```

当进入这个方法说明共享资源已经被其他线程获取了,当前节点之前的节点都不会变化了所以在这个时候修改```pred```是安全的.  

#### 独占模式释放锁

上面我们讲了独占模式下如何获取锁,下面介绍如何释放锁.它的入口方法为```release(int arg)```.

```java
    public final boolean release(int arg) {
        //tryRelease根据自己的需求实现
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

解锁过程相对比较简单,核心的方法就是```unparkSuccessor```.

```java
    //唤醒节点
    private void unparkSuccessor(Node node) {
        //获取当前节点的等待状态
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        //获取当前节点的下一个节点
        Node s = node.next;
        //如果下一个节点为空或者取消了,就找到队列最开始的有效节点
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从队列的尾部向头部开始找,找到队列第一个有效状态(waitStatus < 0)的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //如果找到了,则唤醒封装在s中的线程.
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

可以发现解锁唤醒线程的核心就是调用```LockSupport.unpark(s.thread)```.而获取锁时线程的等待是通过```LockSupport.park(this)```实现的.  
上面代码中还有个一个关键点就是如果当前节点的后继节点为空或者后继节点无效则需要找到一个节点唤醒,这里面为什么是从尾节点往前找而不是往后找?  
在入队的操作中,我们的代码如下:

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                //可能这个还未执行
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

节点加入到尾节点会先将队列之前的```tail```节点设置为当前当前节点的```pred```,然后通过一个```CAS```操作将当前节点设置成尾节点.如果设置当前尾节点成功那么```node.pred = pred```和```compareAndSetTail(pred, node)```就可以看做是一个原子操作了.但是```pred.next = node```这个操作并不能保证,很可能这个还未执行.同时我们在取消节点时,也是修改```next```并未修改```pred```.所以在这里才会从```tail```往```head```方向查找.

## 小结

上面是我对AQS中独占模式获取锁和释放锁的所有分析.可能分析的不是那么精准到位.后面还有两部分内容关于```共享模式锁的获取和释放```还有```Condition实现```.这两部分内容会通过两篇文章讲述,欢迎关注查看后续更新内容.
