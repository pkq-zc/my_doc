# (5)synchronized关键字

## 引出

下面代码段,使用两个线程执行,每个线程执行1000次,每次执行i++

``` java
public class App1 implements Runnable{
    static Integer i = 0;

    public void add(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 1000; j++) {
            add();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        App1 app1 = new App1();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app1);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("i = " + i);
    }
}
```

最后i的结果并不是我们想象中的2000.那么导致这种情况的发生主要是因为多个线程,对同一个共享变量进行修改,就可能会引发线程不安全.因为```i++```并不是一个原子操作,它由多个操作组成.要解决这个问题,最简单的方法就是使用```synchronized```修饰```add()```方法.
当使用```synchronized```修饰```add()```方法时,线程在执行```add()```方法时,需要先获得```app1```这个实例的对象锁,只有获得这个锁,才能执行```add()```方法内部的逻辑.所以保证了,在多线程并发的情况下,每次只能有一个线程执行```i++```,这样就保证了线程的安全性.

## 三种应用方式

### 作用在实例方法

当使用```synchronized```修饰实例方法时,它的锁就在当前这个实例上.下面演示一个错误的例子.

```java
public class App2 implements Runnable{
    static Integer i = 0;

    public synchronized void add(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 1000; j++) {
            add();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        App2 app1 = new App2();
        App2 app2 = new App2();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app2);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("i = " + i);
    }
}
```

上面代码中,```add()```方法已经使用```synchronized```,但是最后结果还是错误.因为```app1```和```app2```不是同一实例,t1和t2在执行```add()```方法时,各自获得了```app1```和```app2```实例的锁,而不是同一实例,所以```synchronized```失效.修改代码如下:

```java
public static void main(String[] args) throws InterruptedException {
        App2 app1 = new App2();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app1);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("i = " + i);
    }
```

### 作用在静态方法

当使用```synchronized```修饰静态方法时,其锁就是当前类的class对象锁.

```java
public class App3 implements Runnable{
    static Integer i = 0;

    public static synchronized void add(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 100000; j++) {
            add();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        App3 app1 = new App3();
        App3 app2 = new App3();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app2);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("i = " + i);

    }
}
```

上面代码不管运行多少次,最后的答案都是200000.可以看见,我们使用的是两个不同的实例,分别为```app1```和```app2```,如果是作用在实例方法上,那么一定会有线程安全问题.但是现在是作用于静态方法,是类的class对象锁,所以是线程安全的.如何证明?看下面实例代码:

```java
public class App3 implements Runnable{
    static Integer i = 0;

    public static synchronized void add(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 100000; j++) {
            add();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        App3 app1 = new App3();
        App3 app2 = new App3();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app2);
        Thread t3 = new Thread(() -> {
            for (int j = 0; j < 100000; j++) {
                i++;
            }
        });
        t1.start();
        t2.start();
        t3.start();
        t1.join();
        t2.join();
        t3.join();
        System.out.println("i = " + i);

    }
}
```

新增加一个线程,没有使用```synchronized```修饰```i++```,结果可想而知,可能会小于```300000```.如果上述条件成立,修改代码如下,对```i++```使用```synchronized```修饰,使用```App3.class```做同步对象.

``` java
    public static void main(String[] args) throws InterruptedException {
        App3 app1 = new App3();
        App3 app2 = new App3();
        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app2);
        Thread t3 = new Thread(() -> {
            for (int j = 0; j < 100000; j++) {
                synchronized (App3.class){
                    i++;
                }
            }
        });
        t1.start();
        t2.start();
        t3.start();
        t1.join();
        t2.join();
        t3.join();
        System.out.println("i = " + i);
    }
```

经过测试,每次结果都是```300000```,说明```synchronized```作用在静态方法上,同步的锁为类class对象锁.

### synchronized同步代码块

有些时候我们并不需要把整个方法都同步,我们只有一小部分代码需要同步.如果整个方法使用```synchronized```同步,能保证线程安全,但是效率比较低.所以我们只需要同步代码块就行.使用方法入下代码所示:

```java
public class App4 implements Runnable{
    static int i = 0;
    private final Object o;

    public App4(Object o) {
        this.o = o;
    }

    public void add(){
        System.out.println("不需要同步的费时代码块1");
        synchronized (o){
            i++;
        }
        System.out.println("不需要同步的费时代码块2");
    }

    @Override
    public void run() {
        for (int j = 0; j < 1000; j++) {
            add();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        App4 app1 = new App4(lock);
        App4 app2 = new App4(lock);

        Thread t1 = new Thread(app1);
        Thread t2 = new Thread(app2);

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("i = " + i);

    }
}
```

从代码看出，将synchronized作用于一个给定的实例对象lock，即当前实例对象就是锁对象.每次线程想进入代码块中都必须先获取lock实例的对象锁.

## 可重入性和不可中断性

### 可重入性

可重入性是指同一线程在外层函数获得锁之后,内层函数可以直接再次获取该锁.而不需要释放当前锁,再去重新获取该锁.

``` java
public class App5 {
    private final Object o;

    public App5(Object o) {
        this.o = o;
    }

    public void method1(){
        synchronized (o){
            System.out.println(Thread.currentThread().getName()+":执行method1");
            method2();
        }
    }

    public void method2(){
        synchronized (o){
            System.out.println(Thread.currentThread().getName()+":执行method2");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        App5 app = new App5(new Object());

        Thread t1 = new Thread(app::method1);
        Thread t2 = new Thread(app::method1);
        Thread t3 = new Thread(app::method1);

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();
    }
}
```

上面代码,```method1()```的同步代码块中,再次调用```method2()```,它并不需要先释放同步对象锁再次获取.

### 不可中断

一旦锁已经被别人获得,如果我还想获得同一个锁,我只能等待或者阻塞,直到别人释放这个锁.如果别人永远不释放锁,那么我只能永远等待下去.

## 总结

```synchronized```在编译后会在同步块前后分别形成```monitorenter```和```monitorexit```这两个字节码指令,这两个指令都需要一个引用类型的对象来指定加锁和解锁的对象.如果java程序中明确指定了对象参数,那就是这个对象的引用.如果没有明确指定,那就看```synchronized```修饰的是实例方法还是类方法,去取对应的对象的实例或```Class```对象来作为锁.

|类型|锁定对象|
|:---|:---|
|指定对象|指定对象|
|实例方法|当前实例|
|类方法|当前类class对象|

根据虚拟机的规范,在执行```monitorenter```时首先需要尝试获取对象的锁.如果该对象没有被锁定,或者已经锁定则可以直接获取,把锁计数器+1.相应的在执行```monitorexit```时会将锁计数器-1.当计数器为0时锁被释放.如果尝试获取锁失败,那当前线程将被阻塞(BLOCK)直到另外的线程将锁解除.
