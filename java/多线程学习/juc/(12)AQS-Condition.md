# AQS-Condition

在之前的文章中已经介绍了独占模式和共享模式获取资源以及使用资源的分析了,现在开始介绍```Condition```.```Condition```是用来替换监视锁的```wait/notify```,但是它比```wait/notify```这种机制更加灵活.在我之前写的[ReentrantLock](https://www.jianshu.com/p/3f00c9588c32)中介绍过如何使用.如果还不知道它是如何使用的可以先参照该文章中对```Condition```使用的介绍再来看本文.本文主要是将如何在AQS中是如何来实现的.

## Condition Queue

在前面的文章中说过AQS中存在一个变种CLH实现的队列,而AQS中的```Condition```其实也就是一个队列,该队列是一个单向链表实现的队列,在这里我就叫他```Condition Queue(条件队列)```了.在AQS中有一个```ConditionObject```内部类,该类就实现了```Condition```接口.在该类中有几个比较重要的属性.

- ```firstWaiter```:头节点,用来指示队列的头部.
- ```lastWaiter```:尾节点,用来指示队列的尾部.

链表中的节点就是就是我们之前介绍的```Node```.节点中有一个属性```nextWaiter```在我们之前的介绍中它是用来标记节点是```SHARED```还是```EXCLUSIVE```.但是在同步队列中它的作用是用来指示下一个节点,这样就形成了一个单向链表了.简单的数据结构就如下图所示:

![条件队列.jpg](https://i.loli.net/2020/07/27/s5gXZ7CGA1VWUT4.png)

上图中:

- ```thread```:用来保存正在等待条件的线程.
- ```waitStatus```:用来保存节点的状态.
- ```nextWaiter```:用来指示下一个节点.

而构建同步队列中关键属性```prev```和```next```在同步队列中并任何作用,我们暂时不需要关注他们,在这里我们需要关系的就是上面几个属性了.我们没创建一个```Condition```其实就是创建了一个新的条件队列.

## await

通过上面对```Condition Queue```的大致了解我们现在开始从```await```方法开始分析.

```java
        public final void await() throws InterruptedException {
            //判断线程是否已中断
            if (Thread.interrupted())
                throw new InterruptedException();
            //将当前线程封装成Node添加到条件队列并返回Node
            Node node = addConditionWaiter();
            //完全释放当前Node所携带的资源
            int savedState = fullyRelease(node);
            //中断类型
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                //挂起线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

上面是调用```await```方法时的内容,我们先分析到```LockSupport.park(this);```,即当前线程挂起这个地方,后续内容稍后分析.首先它先判断调用```await```方法的线程是否中断,如果中断了直接抛出中断异常.接着它调用```addConditionWaiter()```方法将当前线程添加到同步队列,我们来看这个方法内部如何实现.

```java
        private Node addConditionWaiter() {
            //使用临时变量t保存旧的尾节点
            Node t = lastWaiter;
            //如果尾节点不为空且尾节点的waitStatus不是CONDITION,则删除无效节点.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                //删除无效节点后再次使用t保存尾节点
                t = lastWaiter;
            }
            //将当前线程封装成Node,状态为:CONDITION(-2)
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                //如果没有尾节点,说明队列是空的,初始化头节点
                firstWaiter = node;
            else
                //队列已经初始化过了,将旧的尾节点的下一个节点属性指向当前创建的节点(其实就是将当前节点添加到尾部)
                t.nextWaiter = node;
            //设置当前节点为新的尾节点
            lastWaiter = node;
            //返回新创建的节点
            return node;
        }
```

上面这个方法内容注释已经讲的很清楚了.主要任务就是将当前线程封装成```Node```,然后将这个封装好的```Node```添加到队列的尾部.需要注意的是,在将当前节点添加到尾节点前,需要找到有效的尾节点,并将无效的节点从条件队列中删除.

```java
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

上面的代码其实就是删除链表中无用节点的实现,该方法是从链表的头部开始往后开始删除无效节点,主要就是解除节点```nextWaiter```的引用和设置```firstWaiter```和```lastWaiter```引用来实现.这就是一个简单单向链表的删除操作在这里不多说了.  

这是后调用```await()```方法已经走到了```int savedState = fullyRelease(node);```这里了.这个方法主要就是用来```完全释放共享资源的```.完全释放的意思就是一次性释放所有的共享资源.因为对于可重入锁而言,它是可以多次获取同步资源的.我们来看看这个方法如何实现:

```java
    final int fullyRelease(Node node) {
        //用来标记中间是否出现异常
        boolean failed = true;
        try {
            //获取当前的共享资源数
            int savedState = getState();
            //释放指定数目共享资源
            if (release(savedState)) {
                //释放成功,将失败标记标记成false,并返回释放资源的个数
                failed = false;
                return savedState;
            } else {
                //释放资源失败
                throw new IllegalMonitorStateException();
            }
        } finally {
            //如果失败需要将当前节点设置成CANCELLED便于清除无效节点
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

代码中的注释已经展示了代码的执行逻辑,关于```release(savedState)```方法在之前的[独占模式释放锁](https://www.jianshu.com/p/8966cf43ea86)中已做解释,这里不再做讲解了.如果释放锁失败,则抛出```IllegalMonitorStateException```异常.这里正好验证了我们说的```在调用await()方法前必须先获取锁,否则会报异常```的说明.如果节点释放锁失败我们需要将该节点状态设置为```CANCELLED```,便于我们后面将它删除.这也就是我们之前在调用```addConditionWaiter```时为什么需要检查尾节点是否已经取消了的操作.  
接下来通过```isOnSyncQueue(node)```即当前节点是不是在同步队列中判断是不是要挂起线程.可以看见它是一个while循环,第一次```isOnSyncQueue(node)```返回的则是```false```,则会调用```LockSupport.park(this)```将当前线程挂起.我们来看```isOnSyncQueue```方法内部如何实现.

```java
    //判断当前节点是不是在同步队列
    final boolean isOnSyncQueue(Node node) {
        //通过CONDITION或者prev判断
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //通过后继节点判断
        if (node.next != null)
            return true;
        return findNodeFromTail(node);
    }
    //从尾节点往前开始找该节点是不是在同步队列
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                //存在
                return true;
            if (t == null)
                //已到遍历完了还是没找到
                return false;
            //设置为前置节点继续找(从尾到头)
            t = t.prev;
        }
    }
```

为什么要判断当前节点是不是在同步队列呢?在后面的```signal```分析中会讲为什么.现在你只需要知道调用```signal```之后位于条件队列中的节点会被转移到同步队列.对于上面两个判断条件我这里做一下解释:  

- ```node.waitStatus == Node.CONDITION || node.prev == null```:因为位于同步队列中节点它的状态不可能为```CONDITION```,而节点添加到同步队列的方式都是从尾节点开始插入的,而且插入都是先设置pre节点,然后再通过CAS的方式将自己设置成尾节点.如果还是不明白请看同步队列节点入队相关代码.

- ```node.next != null```:在条件队列中,```next```是没有用到的,所以它的默认值应该为```null```.  

经过上面分析,线程调用```await```之后会在方法内部中调用```LockSupport.park(this)```使线程挂起.后面的代码分析我们在讲完```signal```之后继续.

## signal

```java
        public final void signal() {
            //判断当前持有锁的线程是不是当前线程
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            //如果头节点不是null,则调用doSignal
            if (first != null)
                doSignal(first);
        }
```

```isHeldExclusively()```这个方法在AQS中并没有实现,它是用来判断当前持有锁的线程是不是当前线程.这个方法需要同步器自己去实现.如果当前线程没有持有锁,则同样抛出```IllegalMonitorStateException```异常.这个里面最重要的方法就是```doSignal(first)```,我们来看这个方法如何实现.  

```java
        private void doSignal(Node first) {
            do {
                //将头节点指向当前节点的下一个节点,然后判断新头节点是不是空
                if ( (firstWaiter = first.nextWaiter) == null)
                    //如果新头结点是空说明条件队列中已经没有数据了,将lastWaiter设置为null
                    lastWaiter = null;
                //将当前节点的下一个节点设置成null,相当于当前节点是一个单独节点,从队列中删除了
                first.nextWaiter = null;
                //将first节点移到同步队列
                //如果转移失败则重复上面的操作
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

        //将节点从条件队列转移到同步队列
        final boolean transferForSignal(Node node) {
            //如果节点在同步队列状态无效,说明该节点被取消了直接返回fasle
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;
            //将该节点添加到同步队列并返回节点的前驱节点(此处是以独占模式入队的)
            Node p = enq(node);
            int ws = p.waitStatus;
            //如果当前node中的线程无法通过AQS唤醒则直接在这里唤醒
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
            return true;
        }
```

上面的```doSignal(Node first)```主要的操作就是将条件队列中第一个未被```cancalled```的节点"唤醒".首先它通过```firstWaiter = first.nextWaiter```和```first.nextWaiter = null```将头节点移出,然后使用```transferForSignal(first)```将它转移到同步队列.如果失败则一直重复上述操作,直到转移成功或者条件队列为空```(first = firstWaiter) == null```.  
```transferForSignal```方法将节点移动到同步队列首先判断节点是不是有效,无效则直接返回失败.如果节点有效就将节点添加到同步队列(```Node p = enq(node)```)并返回节点的前置节点.接着判断前置节点是不是有效(```ws > 0```)或者设置前节点状态为```SIGNAL```是否成功来决定是不是要唤醒当前节点.如果```ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)```成立说明当前node指向的节点无法通过AQS的队列唤醒,所以在这里才会直接唤醒.而源码中也解释了```in which case the waitStatus can be transiently and harmlessly wrong```.因为即使唤醒了节点并不意味着它就获取到了锁,它还是需要再去抢锁,没有抢到也没有什么问题.  

## signalAll

通过对上面```signal```的了解后,再看```signalAll```就很轻松了.与```signal```不同的是```signal```一次只会将一个节点添加到同步队列,而```signalAll```会清空条件队列,然后将队列中的节点一个一个添加到同步队列.

```java
        private void doSignalAll(Node first) {
            //清空队列
            lastWaiter = firstWaiter = null;
            //一个个将节点添加到同步队列,直到所有节点都添加完了
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```

## await()中LockSupport.park(this)后分析

```java
        public final void await() throws InterruptedException {
            //判断线程是否已中断
            if (Thread.interrupted())
                throw new InterruptedException();
            //将当前线程封装成Node添加到条件队列并返回Node
            Node node = addConditionWaiter();
            //完全释放当前Node所携带的资源
            int savedState = fullyRelease(node);
            //中断类型
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                //挂起线程
                LockSupport.park(this);
                //判断中途是否有中断
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

接着分析之前```await()```部分分析,线程在```LockSupport.park(this)```后继续执行.而线程之所以被唤醒执行只可能是```当前线程被中断```或者```其他线程调用了unpack```方法,不管是哪种情况导致的唤醒,最后该节点都将离开```条件队列```进入```同步队列```,然后使用```acquireQueued```方式获取到锁最后才能退出```await```方法.我们先知道是这样的,然后通过中断时间点作为分析起点开始分析.  

从源码可以知道,当线程被唤醒后就开始调用```checkInterruptWhileWaiting```来判断线程是否被中断了.这个方法不仅能分析出线程是否被中断,还指示了再await()方法退出前需要做什么:

- ```0```:代表整个过程没有中断发生.
- ```REINTERRUPT```:代表在退出await()方法前再次自我中断即可.
- ```THROW_IE```:代表在退出await()方法前需要抛出中断异常.

但是看完你肯定在想为什么要这么做呢?  
首先你要清楚的一件事是```acquireQueued```去获取锁这种方式并不会响应中断,这里所指的是在获取锁的过程中中断和不中断并不会有什么区别,唯一有区别的是线程的中断状态不一样,而在该方法内部并有根据中断状态做什么特殊操作,仅仅只是修改了返回值的中断状态.关于```acquireQueued```方法的内部详情,请参照之前写的```独占模式获取锁```相关内容.  
正是因为```acquireQueued```不会对中断做特殊处理,我们需要判断中断发生的时间点.我们根据当前节点所在队列可以分为两种情况:  

- ```在条件队列```:说明在中断发生时,它还没有被signal属于正常的等待状态中,此时被中断将导致正常的线程等待状态被中断,进入到同步队列抢锁.因此我们在退出await前需要抛出中断异常用来代表是因为中断而导致线程醒来.而这种情况刚好对应的就是```THROW_IE```.
- ```在同步队列```:说明在中断发生时,它已经被signal,这个时候线程已经位于同步队列了.所以这个时候即便是发生了中断,我们都将忽略它,所以仅仅只是在await退出前再次自我中断即可.而这种情况对应的就是```THROW_IE```.

根据上面的分析,我们就按照线程中断的情况做分析:

### 在条件队列中被中断

```java
        //检查是否中断
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
        final boolean transferAfterCancelledWait(Node node) {
            if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
                enq(node);
                return true;
            }
            while (!isOnSyncQueue(node))
                Thread.yield();
            return false;
        }
```

在当前情况下```Thread.interrupted()```的值为```true```,接下来进步一判断中断调用```transferAfterCancelledWait```.而当前节点在是不是在条件队列.因为如果当前节点在同步队列它的状态一定是```CONDITION```.所以```compareAndSetWaitStatus(node, Node.CONDITION, 0)```将会成功.接下来则是调用```enq(node)```方法将当前节点放入到同步队列,需要注意的是这个时候该节点并没有从等待队列删除.  
接下来回到```await()```方法内部,因为```(interruptMode = checkInterruptWhileWaiting(node)) != 0```条件不成立,退出while循环.接下来执行的代码如下:

```java
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
if (node.nextWaiter != null)
    unlinkCancelledWaiters();
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

上面的代码就是我们之前说的线程被唤醒后会```acquireQueued```方法去抢锁,而这里的```savedState```就是我们之前释放资源的个数,关于该方法在之前的文章中有介绍过这里就不再做解释了,该方法最后的返回值是线程是否中断.在我们当前分析的情况下,```interruptMode != THROW_IE```不会成立,程序继续向下执行.  
我们之前说过,在```transferAfterCancelledWait```中只是将当前节点加入到同步队列,但是并没有对条件队列中的节点进行删除,所以接下来```node.nextWaiter != null```成立后要做的事情就是将条件队列中的节点删除掉.这个方法在上面也介绍过这里不做过多解释.  
最后如果线程被中断过,则进行```reportInterruptAfterWait(interruptMode)```处理.

```java
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```

经过上面整个过程分析,当线程还在条件队列中就被删除了(即还没有被signal),会做下面几件事:

- 将当前节点添加到同步队列
- 通过```acquireQueued```获取锁,没有获取到再次挂起
- 获取到锁之后,将之前的节点从条件队列删除
- 根据中断类型做中断异常处理.

### 在同步队列中断

与```在条件队列中断```不同的是,```在同步队列中断```这种情况要简单一些,主要差别就在于中间省略了```enq```(即从条件队列同步转移到同步队列)的操作.

```java
        final boolean transferAfterCancelledWait(Node node) {
            if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
                enq(node);
                return true;
            }
            while (!isOnSyncQueue(node))
                Thread.yield();
            return false;
        }
```

该情况中```compareAndSetWaitStatus(node, Node.CONDITION, 0)```不成立,即意味着节点已经添加到了同步队列.其实这个时候并不能保证一定添加到同步队列了,因为可能正在添加到同步队列.

```java
    final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

在```transferForSignal```方法中,可能因为其```compareAndSetWaitStatus(node, Node.CONDITION, 0)```执行成功,导致```transferAfterCancelledWait```中的```compareAndSetWaitStatus(node, Node.CONDITION, 0)```执行失败,这就是我们说的可能正在同步队列.这个时候处理也很简单,就是自旋判断节点是不是成功添加到同步队列即可.  
后面的操作与之前的并没有太大区别.最后就是在```reportInterruptAfterWait```根据中断情况做一个自我中断的操作.  

## 小结

通过对```Condition```的分析,最后重点也就是下面几个点:

- ```Condition```其实就是一个单向链表构成的队列.
- 线程调用```await```就将自己添加到条件队列,然后释放自己带的锁,如果没有获取到锁就抛出异常.
- 线程调用```signal```就是从条件队列中获取一个节点,然后将该节点添加到同步队列让它抢锁.