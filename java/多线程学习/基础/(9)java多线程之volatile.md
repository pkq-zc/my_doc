# (9)volatile

## 可见性

当一个变量使用volatile修饰,那么该变量具有在所有线程中的可见性.这里面的可见性是指当一个线程对使用volatile修饰的变量进行更新,在其他线程中能立马获取到更新之后的值,这一点是其他普通变量做不到的.  

```java
public class App1 {
    boolean flag = true;
    public static void main(String[] args) throws InterruptedException {
        App1 app1 = new App1();
        Thread t1 = new Thread(() -> {
            long i = 0;
            while (app1.flag){
                i++;
            }
            System.out.println("out loop. i= "+i);
        });
        t1.start();

        TimeUnit.SECONDS.sleep(1);
        app1.flag = false;
        System.out.println("修改flag值为false");
    }

}
```

上面代码中,通过一个状态变量flag控制t1线程循环增加i的值,而main线程在1s之后修改循环控制变量flag的值,让t1退出循环并打印n的值.但是因为普通变量的可见性,导致t1无法获取到flag更新后的值,导致程序陷入死循环中无法退出.我们只需要使用```volatile```修饰flag的值,它就可以让线程t1看到已经被修改的flag变量值.  
但是经常被误解的是,volatile变量在并发下是线程安全的.这个观点是错误的.

``` java
public class App2 {
    private static volatile int sum = 0;
    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = () -> {
            for (int i = 0; i < 1000; i++) {
                sum++;
            }
        };

        Thread t1 = new Thread(runnable, "t1");
        Thread t2 = new Thread(runnable, "t2");
        Thread t3 = new Thread(runnable, "t3");

        t1.start();
        t2.start();
        t3.start();
        t1.join();
        t2.join();
        t3.join();

        System.out.println("sum = " + sum);
    }
}
```

例如上面的代码,它最后的结果大概率不是3000.因为```i++```并不是一个原子操作.执行该语句首先要获取i的值,然后执行i+1操作,然后将计算结果赋值给i.所以,即使使用volatile修饰,在并发情况下计算使用volatile修饰的变量无法保证线程安全.在无法满足下面两条规则的情况下,我们还是需要通过同步块或者加锁才能保证线程安全.

- 运算结果并不依赖当前值,或者能够确保只有一个线程会修改变量的值.
- 变量不需要和其他状态变量共同参与不变约束

## 禁止指令排序优化

普通变量仅仅只会保证该方法执行过程中所有依赖赋值结果的地方才能获取真正的结果,而不能保证变量赋值的操作的执行顺序与代码中的书写顺序一致.在DCL单例模式中,很好的体现了volatile禁止指令排序优化.

``` java
class Singleton {
    private static  Singleton singleton;

    private Singleton(){}

    public static Singleton getInstance(){
        if (singleton == null){
            synchronized (Singleton.class){
                if (singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

上面这个代码看上去没什么问题,而且在大多数情况下多线程并发也很难出现问题.但是它还是存在一个问题.```singleton = new Singleton()```简单的来说需要经过三个步骤,1)分配内存空间,2)初始化,3)将内存地址赋值给对应的引用.因为存在指令排序优化,当然不是乱优化.这里面可以是1->2->3,也可以是1->3->2.如果顺序是1->3->2的情况,singleton指向了一个未被初始化的对象,另外一个线程在执行第一个```singleton == null```可能读取到singleton的值不为空,那么就可能返回一个还未初始化的对象,如果使用该对象,就可能导致错误.如果加了volatile修饰singleton变量,禁止对指令进行排序优化,那么就不会出现这个问题.这也就是为什么DCL单例模式需要使用volatile来修饰变量的原因.

## JMM对volatile的规定

在java内存模型中,有下面的规定.假设线程T和变量V.

- 只有当线程T对变量V执行的前一个动作是load的时候,线程T才能对变量V执行use动作;并且,只有当线程T对变量V的后一个动作是use的时候,线程T才能对变量V执行load动作.简单的说这条规则要求在工作内存中,每次使用V前必须从主内存中刷新最新的值,用于能看见其他线程对V所做的修改后的值.
- 只有当线程T对变量V执行的前一个动作是assign的时候,线程T才能对变量执行store动作;并且只有当线程T对变量V的最后一个动作是store的时候,线程T才能执行assign动作.简单的讲就是这条规则要求在工作内存中,每次修改完变量的值都必须同步回主内存中,用于保证其他线程可以看见自己对变量V所做的修改.
- 加入A动作之前必须关联P和F动作,B动作之后必然伴随G和Q动作.如果存在A优先于B,那么P或者F,必然优先于B,G,Q等动作.这里的动作可以是T对V实施use或者assign等等.这个规则就保证了volatile修饰的变量不会被指令重排优化.
