# (11)CAS

## 什么是CAS

简单的说,```CAS```就是```compare and swap```,翻译成中文就是比较与交换.在java多线程中,我们可以使用锁,synchronized等来保证多线程下共享数据的安全,这些方式都算是```悲观锁```,而```CAS```算是```乐观锁```.而所谓的乐观锁和悲观锁只是我们人为的一种划分.悲观锁认为每次并发都认为别人会修改共享数据,使用锁和同步代码块等方式使得一次只能有一个线程去操作共享数据.乐观锁则认为每次并发都不会有其他人修改共享数据,只是在更新的时候才去判断数据是否被修改过.  
```CAS```是一种无锁算法,它的核心思想就是比较和交换.CAS涉及三个操作数:

- 需要读写的内存值V(就是内存中现在的值)
- 进行比较的值A(之前从内存中读取的值)
- 拟定写入的值B(更新的新值)

当且仅当V的值和A的值一样时,CAS才会使用原子的方式使用新值B来更新V的值(这里面的比较和设置新值是一个原子操作).如果上面的相等条件不成立,那么会进行一个自旋操作,即重新尝试.

## 优缺点

### 优点

- 在并发量不是特别大的时候,它的性能要比使用synchronized和锁的效率要高.

### 缺点

- ```循环时间长开销大```:在高并发的情况下,CAS失败次数增加,导致尝试次数增加.
- ```只能保证一个共享变量的原子操作```:当使用一个变量时,我们可以循环使用CAS的方式保证原子操作.但是如果是多个变量时,这个时候并不能保证还是原子操作.
- ```ABA问题```:如果内存的初读值为A,然后另外一个线程将其值改成B,然后又改回A.那么这个时候使用CAS判断是成功的.实际上真正的值已经经过了一次修改.所以在使用时,也需要考虑清楚```ABA```问题会不会影响业务数据.

### java中是实现

在jdk中主要提供了四类CAS实现,分别为:

1. 基本类型
    - ```AtomicBoolean```[原子更新布尔类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App1.java)
    - ```AtomicInteger```[原子更新整型类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App1.java)
    - ```AtomicLong```[原子更新长整型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App1.java)
2. 数组类型
    - ```AtomicIntegerArray```[原子更新整型数组](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App2.java)
    - ```AtomicLongArray```[原子更新整型数组](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App2.java)
    - ```AtomicReferenceArray```[原子更新引用类型数组](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App2.java)
3. 引用类型
    - ```AtomicReference```[原子更新引用类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App3.java)
    - ```AtomicReferenceFieldUpdater```[原子更新引用类型字段类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App4.java)
    - ```AtomicMarkableReference```[原子更新带有标记的引用类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App6.java)
4. 字段类型
    - ```AtomicIntegerFieldUpdater```原子更新整型的字段的更新器
    - ```AtomicLongFieldUpdater```原子更新长整型的字段的更新器
    - ```AtomicStampedReference```[原子更新带有版本号的引用类型](https://github.com/pkq-zc/current/blob/master/src/main/java/com/buydeem/atomic/App7.java)

### java解决ABA问题

``` java
public class App5 {
    private static AtomicInteger i = new AtomicInteger(1);

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            String name = Thread.currentThread().getName();
            //获取值
            int old = App5.i.get();
            System.out.println(name+" ==> i ="+ old);
            //  休眠1秒
            try {
                Thread.sleep(1000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //更新值
            boolean success = i.compareAndSet(old, 100);
            if (success){
                System.out.println(name+" CAS更新操作成功 ==> " +i.get());
            }else {
                System.out.println(name+" CAS更新操作失败 ==> " +i.get());
            }
        }, "t1");

        Thread t2 = new Thread(()->{
            String name = Thread.currentThread().getName();
            try {
                Thread.sleep(500L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //修改成2
            i.set(2);
            System.out.println(name+" ==> i = "+i.get());
            //修改成1
            i.set(1);
            System.out.println(name+" ==> i = "+i.get());
        },"t2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

最后打印结果如下:

```java
t1 ==> i =1
t2 ==> i = 2
t2 ==> i = 1
t1 CAS更新操作成功 ==> 100
```

上面是ABA问题的示例代码.当线程t1在准备更新值时(此时值为1),因为线程睡眠t1暂停执行后面的代码.t2开始执行,t2先将值改为2,然后t2又将值改回1.当线程t1睡眠结束,使用```compareAndSet```更新值为100时,发现预期值没有发生改变,认为没有其他线程修改过该数据,那么这就出现了ABA的问题.  
在jdk中提供了```AtomicMarkableReference```和```AtomicStampedReference```来解决这个问题.一个是通过boolean来判断值是否改变过,另一个通过一个整型的版本号来判断是否改变过,这样更新的时候不仅仅只看值是否发生过改变,还需要判断更新标记或者版本号是否发生过改变来判断值是否有过改变.

## 底层实现

通过查看源码,发现底层最后调用的都是Unsafe类提供的方法实现.该类主要提供了一些执行级别低,不安全的操作.例如可以直接访问JVM以外的内存,可以直接操作内存空间等.归根结底它就是通过一条CPU原子指令(cmpxchg指令)来实现的,该指令就是CAS的核心思想.具体Unsafe提供哪些功能,可以参考下面链接[Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)
