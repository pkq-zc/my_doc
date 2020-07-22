# CLHLock实现

## 简介

在AQS的源码中有介绍到一种叫```CLH```的队列(AQS中使用的是一种变种).```CLH```它是一种基于单向链表实现的队列,即后一个节点保存前一个节点的引用形成单向链接.我们可以使用```CLH```来实现一个高效的自旋锁.每个线程请求锁时,都先判断它的前节点是否需要锁,如果前节点需要锁则自己自旋等待.如果前节点不需要锁则获取锁成功.  
正是因为它的这种实现方式,它具有先来先获取的公平性.而它的核心基于一个CAS操作来实现具有很高的性能.但因为其自旋等待,所以不太适合占用锁时间过长的场景,因为自旋是会需要消耗CPU资源的.

## 简单实现

```java
/**
 * 声明锁接口
 */
public interface Lock {
    /**
     * 获取锁
     */
    void lock();

    /**
     * 解锁
     */
    void unlock();
}

/**
 * 定义Node
 */
public class CLHNode {
    /**
     * 使用volatile修饰保证可见性.
     */
    volatile boolean locked;
}

/**
 * CLHLock实现
 */
public class CLHLock implements Lock{
    /**
     * 保存当前节点,一个线程一个
     */
    private final ThreadLocal<CLHNode> node;
    /**
     * 尾节点.所有线程共享一个.支持CAS操作
     */
    private final AtomicReference<CLHNode> tail;
    /**
     * 父节点.一个线程一个
     */
    private final ThreadLocal<CLHNode> pred;

    public CLHLock() {
        this.node = ThreadLocal.withInitial(CLHNode::new);
        this.tail = new AtomicReference<>(new CLHNode());
        this.pred = new ThreadLocal<>();
    }

    @Override
    public void lock() {
        //获取当前节点
        CLHNode current = node.get();
        //将当前节点设置成需要锁
        current.locked = true;
        //将当前节点设置成尾节点,CAS操作
        CLHNode currentPred = tail.getAndSet(current);
        //设置当前节点的前节点
        pred.set(currentPred);
        //查看当前节点是否需要锁,如果需要则自旋等待
        while (currentPred.locked){

        }
        //获取锁成功
        System.out.println(Thread.currentThread().getName()+":获取到锁");
    }

    @Override
    public void unlock() {
        //获取当前节点
        CLHNode current = node.get();
        //将当前节点locked设置成false,它后面的节点将会推出自旋
        current.locked = false;
        //将当前节点设置成前节点
        node.set(pred.get());
    }
}

/**
 * 使用测试
 */
public class App {

    public static void main(String[] args) {
        Lock lock = new CLHLock();

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                lock.lock();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                lock.unlock();
            }
        };

        Thread t1 = new Thread(runnable);
        t1.setName("t1");
        t1.start();

        Thread t2 = new Thread(runnable);
        t2.setName("t2");
        t2.start();

        lock.lock();
        lock.unlock();

    }
}
```

- ```获取锁```:每个线程获取锁时都自己节点的```locked```设置成```true```代表自己需要锁,然后将自己节点设置成尾节点```tail```(这里使用```AtomicReference```来实现一个```CAS```操作),并获取到当前节点的前置节点.接着获取的前置节点设置成自己的前置节点.然后就是判断通过前置节点的```locked```属性来判断前置节点是否需要锁.这里需要注意的是```locked```属性需要使用```volatile```修饰来保证其他线程修改它的值而当前线程能感知到该值的变化.如果去掉```volatile```可能导致当前线程一直自旋无法获取到锁.
- ```释放锁```:释放锁就比较好理解了,直接获取到当前节点(这就是为什么使用ThreadLocal来保存当前节点的值了).然后将当前节点的```locked```设置成```false```,而这个操作将会使该节点的后置节点退出自旋获取到锁.

关于解锁操作中的```node.set(pred.get())```.它主要是为了防止获取锁解锁之后再次获取锁无法获取锁的bug.如果去掉该行代码,第一次获取锁当前节点已经是```tail```节点了,然后再次获取锁.再次将当前节点作为尾节点.即当前节点的头节点和当前节点是同一个节点.如果这个时候设置```locked```为```true```,这将导致当前节点再判断前节点是否需要锁时一直都是```true```.线程将在获取锁时无限自旋等待.

## 总结

总体来说使用这种方式的有点如下:  

- 简单高效.
- 支持FIFO公平性.
- 自旋等待避免线程切换的性能损耗

同样它也有下面的缺点:

- 不可重入.即获取锁之后必须释放锁才能再次获取锁.
- 锁占用时间过长导致CPU空转.
