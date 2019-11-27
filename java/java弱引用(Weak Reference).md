# java弱引用(Weak Reference)

## 简介

之前在看ThreadLocal源码中,发现其中的静态内部类Entry发现它继承的是一个```WeakReference<ThreadLocal<?>>```,通过查询相关资料发现在java中一共存在四种引用.分别为强引用,软引用,虚引用和弱引用.

## 区别

- ```强引用(Strong Reference)```:通常我们通过new来创建一个新对象时返回的引用就是一个强引用,若一个对象通过一系列强引用可到达,它就是强可达的(strongly reachable),那么它就不被回收
- ```软引用(Soft Reference)```:若一个对象是软引用可达,当前内存充足它不会被回收.
- ```弱引用(Weak Reference)```:若一个对象是弱引用可达,不管当前内存是否充足它都会被回收.

- ```虚引用(Phantom Reference)```:虚引用是Java中最弱的引用,那么它弱到什么程度呢?它是如此脆弱以至于我们通过虚引用甚至无法获取到被引用的对象,虚引用存在的唯一作用就是当它指向的对象被回收后,虚引用本身会被加入到引用队列中,用作记录它指向的对象已被销毁

## 示例

``` java
public class App1 {

    public static void main(String[] args) {
        //创建实例
        Student tom = new Student("tom");
        //设置为弱引用
        WeakReference<Student> weakReTom = new WeakReference<>(tom);
        //获取弱引用对象
        System.out.println(weakReTom.get());
        //去掉tom实例强引用
        tom = null;
        //调用GC
        System.gc();
        //再次获取弱引用对象
        System.out.println(weakReTom.get());

    }
}

@Data
@AllArgsConstructor
class Student{
    private String name;

    @Override
    protected void finalize() throws Throwable {
        //垃圾回收前会调用该方法
        System.out.println(this.toString()+"被回收了");
        super.finalize();
    }
}
```

启动时添加VM参数```-XX:+PrintGC```,打印结果如下:

```
Student(name=tom)
[GC (System.gc())  3336K->775K(125952K), 0.0017908 secs]
[Full GC (System.gc())  775K->672K(125952K), 0.0051377 secs]
null
Student(name=tom)被回收了
```

上面程序,因为weakReTom为弱引用,当垃圾回收时,会被直接回收.
