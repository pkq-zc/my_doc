# (7)pack和unpark

## 作用

用来创建锁和其他同步类的基本线程阻塞的原语.在查看```AbstractQueuedSynchronizer```(AQS)中,发现其底层就是使用```LockSupport.park()```和```LockSupport.unpark()```实现线程的阻塞和唤醒的.它的作用与Object上的wait和notify很像,但是还是存在一些不同.

## 不需要在同步代码块中调用

之前在```java多线程之wait和notify```中有讲过,使用wait()和notify()必须在同步代码快中才是调用,如果不是则会抛出```IllegalMonitorStateException```异常.但是LockSupport.park()则没有该限制.下面示例代码,如果去掉```synchronized```同步代码块将抛出异常.

```java
public class App1 {
    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();

        Thread t1 = new Thread(() -> {
            synchronized (lock){
                try {
                    lock.wait();
                    System.out.println(Thread.currentThread().getName()+":执行完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "t1");
        t1.start();

        Thread t2 = new Thread(() -> {
            LockSupport.park();
            System.out.println(Thread.currentThread().getName()+":执行完成");
        }, "t2");
        t2.start();

        Thread.sleep(1000L);
        //唤醒t1,t2
        synchronized (lock){
            lock.notify();
        }
        LockSupport.unpark(t2);
    }
}
```

## 调用先后不会导致死锁

在使用wait和notify时,如果线程A先调用了notify,然后线程B再调用的wait没有指定超时时间,那么线程B将一直处于```WAITING```状态,直到有其他线程调用notify或者notifyAll.使用park和unpark则不会导致该情况发生.代码如下:

```java
public class App2 {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        Thread t1 = new Thread(() -> {
            synchronized (lock){
                System.out.println(Thread.currentThread().getName()+":开始执行");
                lock.notify();
            }
            synchronized (lock){
                try {
                    lock.wait();
                    System.out.println(Thread.currentThread().getName()+":执行完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName()+":开始执行");
            LockSupport.unpark(Thread.currentThread());
            LockSupport.park();
            System.out.println(Thread.currentThread().getName()+":执行完成");

        }, "t2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

## 响应中断的不同

使用wait使线程进入```WAITING```或者```TIMED_WAITING```,如果中断线程,线程将抛出中断异常```InterruptedException```.如果使用park使线程进入等待状态,并不会抛出中断异常.

```java
public class App3 {
    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        Thread t1 = new Thread(() -> {
            synchronized (lock){
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    Thread thread = Thread.currentThread();
                    System.out.println(thread.getName()+":响应中断退出");
                }
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            LockSupport.park();
            Thread thread = Thread.currentThread();
            System.out.println(thread.getName()+":响应中断退出");
        }, "t2");

        t1.start();
        t2.start();
        Thread.sleep(1000L);
        System.out.println("t1.getState() = " + t1.getState());
        System.out.println("t2.getState() = " + t2.getState());
        t1.interrupt();
        t2.interrupt();
    }
}
```

## 不可重入

LockSupport很类似于二元信号量(只有1个许可证可供使用),如果这个许可还没有被占用,当前线程获取许可并继续执行;如果许可已经被占用,当前线程阻塞,等待获取许可.如果调用unpark多次释放许可,再多次调用park获取许可只能获取到一次许可.如下面代码,第二次调用park将导致线程永远的等待下去.

```java
public class App4 {
    public static void main(String[] args) {
        Thread thread = Thread.currentThread();
        System.out.println("第一次调用unpark");
        LockSupport.unpark(thread);
        System.out.println("第二次调用unpark");
        LockSupport.unpark(thread);
        System.out.println("第一次调用park");
        LockSupport.park();
        System.out.println("第二次调用park");
        LockSupport.park();
        System.out.println("执行完成");
    }
}
```

上面示例代码永远也不会打印```执行完成```.
