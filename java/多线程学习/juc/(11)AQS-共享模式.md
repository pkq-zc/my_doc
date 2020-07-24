# AQS-共享模式

接之前写的[AQS独占模式](https://www.jianshu.com/p/8966cf43ea86)文章,现在对照独占模式我们开始分析.

## 对比

|独占模式|共享模式|
|:---|:---|
|acquire(int arg)|acquireShared(int arg)|
|acquireInterruptibly(int arg)|acquireSharedInterruptibly(int arg)|
|tryAcquireNanos(int arg, long nanosTimeout)|tryAcquireSharedNanos(int arg, long nanosTimeout)|
|release(int arg)|releaseShared(int arg)|

上面列出了独占模式和共享模式获取锁和释放锁的入口方法.我们对比分析就能很清楚的了解它们之间的不同.

### 共享模式获取锁

同独占模式一样,获取锁的入口方法我们从```acquireShared(int arg)```开始,另外还有两个入口如果有兴趣自己分析即可.

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

上面代码第一步开始调用```tryAcquireShared(arg)```尝试获取共享资源,而在独占模式中是调用```tryAcquire(arg)```.同独占模式一样这个方法也需要同步器自己去实现.这个方法返回值为剩余资源的个数,主要可以分为三种情况:  

- ```0```:获取共享资源成功,但是没有剩余的资源了.
- ```>0```:获取资源成功,还有剩余资源
- ```<0```:获取资源失败.

如果获取资源成功则直接返回了,如果失败了则进入```doAcquireShared(arg)```,而独占模式下则是进入```acquireQueued(addWaiter(Node.EXCLUSIVE), arg)```.如果看了之前的文章的人知道下一步是干嘛了.下一个就是要将自己加入到队列的尾部然后挂起等待.

```java
    private void doAcquireShared(int arg) {
        //将自己添加到队列的尾部,这里在独占模式已分析过了,如果想了解请看之前独占模式相关处
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            //同样自旋
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    //如果自己是老二就有资格可以尝试获取资源
                    int r = tryAcquireShared(arg);
                    //如果资源还有剩余
                    if (r >= 0) {
                        //将自己设置为头节点,并传播(后面会解释传播)
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //尝试获取资源失败,后续操作同独占锁
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

上面的代码与独占模式下整体上并没有太大差别,主要流程如下:

1. 调用```addWaiter```将自己添加到队列的尾部
2. 获取当前节点的前节点,然后看前节点是不是头节点.
3. 如果当前节点的前节点是头节点,自己就有资格去尝试获取资源.如果获取失败了就进行后面操作逻辑.如果尝试获取资源成功了,则会调用```setHeadAndPropagate(node, r)```将自己设置成头节点并往后传播.关于```setHeadAndPropagate```这个方法后面会细讲.
4. 第2步判断当前节点不是老二或者第3步获取资源失败就会进入这个逻辑.这个逻辑主要做两件事:```将前置节点设置waitStatuss设置成-1```和```让当前线程挂起```.这个逻辑与独占锁中的并没有差别.

在第3步中调用```tryAcquireShared```获取到资源后的操作与独占模式中不一样.在独占模式中是调用```setHead```将自己设置成头节点,具体代码如下:  

```java
    //独占模式只会将当前节点设置成头节点
    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```

而在共享模式中,它会调用```setHeadAndPropagate```不仅仅只是将当前节点设置成头节点,还有将当前节点的后继几点继续唤醒,具体代码如下:

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head;
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                //唤醒等待获取共享资源的线程,后面细说.目前知道这个方法就是去唤醒等待的节点
                doReleaseShared();
        }
    }
```

收先要明确为什么能进入这个方法,这个方法中的```node```一定是代表它是当前成功获取了共享资源的线程,```propagate```代表了当前线程获取了共享资源后还剩余的线程数.这个方法一定是当前线程成功获取了共享资源才会进来的.首先它通过一个```h```用来保存旧的```head```节点,然后将自己设置成```head```节点.然后进入后面这一堆的判断.下面我们来分析这一系列的判断

- ```propagate > 0```:当前线程获取资源后发现还有剩余资源,那么这个时候就需要唤醒等待的节点.这个很好理解,因为是共享模式,当资源有多余的时候就唤醒其他等待资源的线程.要不然怎么叫共享呢?
- ```h.waitStatus < 0```:这个代表```老的head节点```后面的节点可以被唤醒.
- ```(h = head) == null || h.waitStatus < 0```:这个代表```新的head节点```后面的节点需要被唤醒.

总结来说就是两点:当```propagate > 0```时说明资源可用所以唤醒节点,而```h.waitStatus < 0```说明不管是新的头结点还是老的头节点只要它的```waitStatus < 0```都需要唤醒节点.  

> 关于```h == null```这个条件我觉得不会生效.因为进入该方法前```addWaiter```已经调用了.CLH队列中至少会存在一个节点所以我觉得这个条件不会生效.目前我也想不出什么情况下```h == null```.  

上面就是整个共享模式下获取资源的整个过程.整体上与独占模式下差别不太大.都是获取到了直接返回执行业务代码,获取失败则进入阻塞.但是最大的不同点在于:```共享模式下获取共享资源成功的情况下同时还会去唤醒等待的线程,而在独占模式下是不会的.```

### 共享模式下释放锁

