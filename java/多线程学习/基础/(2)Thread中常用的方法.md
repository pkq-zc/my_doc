# (2)Thread中常用的方法

## start&run

需要注意的是,```start()```才是启动线程的方法,而```run```只是普通的一个实例方法.

## sleep&yield

```sleep```可以让```当前调用线程```休眠指定时间,但是并不会丢失任何监视器的所属权(后面的文章会讲到这个,目前暂时知道就行).  
```yield```它会让```当前调用线程```让出当前持有的cpu资源.简单的说就是本来你持有cpu资源在干活,然后调用了该方法,然后你交出了手里的cpu资源.接下来你和其他所有等待cpu资源的人等待cpu选择,这个时候有可能你会被再次选中.  

sleep和yield方法需要注意的是,它指的是```当前执行的线程```,它是一个静态方法并不是一个实例方法.如果你通过当前实例去执行该方法,该实例线程并不会进入sleep.

```java
public class App2 {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true){
                    System.out.println("线程执行");
                }
            }
        });
        t.setName("t1");
        t.start();
        t.sleep(100000);
    }
}
```

线程```t1```并不会进入sleep,会一直打印```线程执行```.这是因为进入sleep的是```mian```线程,并不是```t1```线程.这就是之前说的起作用的是```当前执行线程```.

## join

当前线程等待被调用的实例线程执行完之后再执行.

```java
public class App3 {
    public static void main(String[] args) throws InterruptedException {
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
        //t1.join();
        System.out.println(Thread.currentThread().getName()+":t1.start()后执行");
    }
}
```

上面的代码如果把```t1.join()```注释,它的执行结果可能有下面几种情况:

```java
//可能的结果1
main:t1.start()前执行
main:t1.start()后执行
t1:执行完毕
//可能的结果2
main:t1.start()前执行
t1:执行完毕
main:t1.start()后执行
//可能的结果3
main:t1.start()前执行
main:t1.start()后执行
```

当main线程执行完```t1.start()```之后,main线程并不会关心t1线程是否已经开始执行或者是否已经执行完毕,main线程会继续向下执行.这就造成了上面三种可能的结果.而取消```t1.join()```这句注释再次执行程序,它最后的打印结果只会是```结果2```,这是因为```main线程调用t1.join(),main线程会等待t1线程执行完所有逻辑才会继续执行```,这就是上面说的```当前线程等待被调用的实例线程执行完之后再执行```.这里面的当前线程指的就是```main线程```,而被调用的实例线程就是指```t1线程```.后续文章讲到线程通信wait和notify时会讲join的实现原理.

## setDaemon

设置线程为守护线程,该方法必须是在线程启动前调用才会有效.  
首先需要了解什么是守护线程.在java中线程一般就是两种:用户线程和守护线程.当jvm只要存在任意一个用户线程,那么守护线程将继续工作下去.如果最后一个用户线程结束执行,那么所有的守护线程将退出执行.守护线程主要为其他线程提供便利服务.例如最典型的就是GC(垃圾回收线程).

```java
public class App4 {
    public static void main(String[] args) throws InterruptedException {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                while (true){
                    System.out.println("当前时间"+new Date());
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };

        Thread t1 = new Thread(null,r,"t1");
        //t1.setDaemon(true);
        System.out.println("系统启动...");
        t1.start();
        Thread.sleep(5000);//用来确保t1线程开始执行
        System.out.println("系统停止...");
    }
}
```

运行结果如下:

```java
系统启动...
当前时间Tue Jun 30 10:17:21 CST 2020
当前时间Tue Jun 30 10:17:22 CST 2020
当前时间Tue Jun 30 10:17:23 CST 2020
当前时间Tue Jun 30 10:17:24 CST 2020
当前时间Tue Jun 30 10:17:25 CST 2020
系统停止...
当前时间Tue Jun 30 10:17:26 CST 2020
当前时间Tue Jun 30 10:17:27 CST 2020
```

因为t1不是守护线程,所以当main线程执行结束之后,jvm并不会退出,因为其中还有用户线程在执行.如果取消```t1.setDaemon(true)```,设置t1为守护线程,当main线程执行结束之后t1线程退出会停止打印.  
需要注意的是设置守护线程必须是在线程未调用start方法前,如果线程已经启动再调用,将会抛出```IllegalThreadStateException```异常.同时还需要注意的一点就是,守护线程创建的线程默认也是守护线程.

```java
public class App5 {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //打印t1线程事发后是守护线程
                System.out.println(Thread.currentThread().getName()+":"+Thread.currentThread().isDaemon());
                //在线程1中创建线程2
                Thread t2 = new Thread(new Runnable() {
                    @Override
                    public void run() {
                        //打印t2线程事发后是守护线程
                        System.out.println(Thread.currentThread().getName()+":"+Thread.currentThread().isDaemon());
                    }
                });
                t2.setName("t2");
                t2.start();
            }
        });
        t1.setName("t1");
        //设置t1线程未守护线程
        t1.setDaemon(true);
        t1.start();

        Thread.sleep(2000);
    }
}
```

最后的打印结果如下:

```java
t1:true
t2:true
```

## 总结

上面这些方法都是Thread中比较常用的方法,但是还有部分比较重要的方法,这些将会在后面的文章中有介绍.
