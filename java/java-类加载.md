# java-类加载

## 什么是类加载机制

虚拟机把class文件加载到内存,并对数据进行校验,转换解析和初始化,最终形成可以被虚拟机直接使用的java类型.  
java与编译时语言不同,java语言中的类型加载,连接和初始化过程都是在程序运行期间完成的.java语言可以动态扩展的特性也就是基于上面的原理来实现的.例如我们可以写一个接口等到运行时再指定由哪个类实现.我们还可以自己实现一个类加载器,然后通过网络或者其他方式来加载字节码等等.  

## 类的加载过程

类从加载到虚拟机,直到类被卸载出内存一共会经历下面七个过程:  

- ```加载```
- ```验证```
- ```准备```
- ```解析```
- ```初始化```
- ```使用```
- ```卸载```

![类加载过程.jpg](https://i.loli.net/2020/08/04/bDAcLIhe98Ral67.png)  

上面图中的验证,准备和解析合在一起交连接.加载,验证,准备,初始化和卸载这5个阶段顺序是确定的,也就是说类加载过程中这几个过程必须会经历,切顺序是不会调换的.

## 加载

加载是类加载中的一个阶段,在加载过程中虚拟机主要完成下面三件事:  

- 通过类的全限定名来获取定义此类的二进制字节流.
- 将字节流所代表的静态存储结构转化成方法区的运行时数据结构.
- 内存中生成代表这个类的```java.lang.Class```对象,作为方法去这个类的各种数据的访问入口.

## 验证

验证主要是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求,并且不会危害到虚拟机自身的安全.  
例如在java中就无法访问到数组外的数据,将一个对象转换成它并未实现的类型,跳转到不存在的代码行之类的事,如果这么做了那么编译器就会拒绝编译.  

在验证过程中主要包括以下几点:  

- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

## 准备

准备阶段是正式为类变量分配内存并设置类变量初始化值的阶段,这些变量所使用的内存都将在方法区中进行分配.  
这里面的类变量指的是使用```static```修饰的变量,而不包括实例变量.而这里面的初值就是指零值,例如int的初值就是0.  

```java
    //常量(不属于类变量)
    public static final String info = "静态常量";
    //类变量
    public static String name;
    //实例变量
    public Integer age;
```

基本数据类型的零值如下表:  

|数据类型|零值|
|:---|:---|
|int|0|
|long|0L|
|short|(short)0|
|char|'\u0000'|
|byte|(byte)0|
|boolean|false|
|float|0.0f|
|double|0.0d|
|引用类型|null|

## 解析

解析阶段就是虚拟机将常量池内的符号引用替换成直接引用的过程.  

## 初始化

初始化阶段就是执行```<clinit>```方法的过程.该方法又编译器自动收集所有的类变量赋值动作和静态语句块中的语句合并产生,而收集顺序就是由源文件中出现的顺序所决定的.  
需要注意的是:```定义在静态语句块之后的变量,在静态语句块中是可以赋值的,但是不能访问```.

```java
class Test{
    static {
        name = "测试";//编译通过
        System.out.println(name);//编译不通过
    }

    public static String name;
}
```

那么什么时候类加载过程中会执行初始化动作呢?```JVM```中明确了下面5种情况.  

1. 使用new关键字实例化对象,读取和设置一个类的静态字段以及调用类的静态方法时.
2. 使用```java.lang.reflect```包中的方法对类进行反射调用的时候,如果该类没有初始化则需要先初始化.
3. 当初始化一个类的时候,该类的父类还没有初始化会先初始化其父类.
4. 当虚拟机启动时指定的主类虚拟机会先初始化这个类.
5. 当使用JDK1.7的动态语言支持时,如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic,REF_putStatic,REF_invokeStatic的方法句柄,并且这个方法句柄所对应的类没有进行过初始化,则需要先触发其初始化.(该条暂时不懂是什么意思)  

```java
public class App1 {
    public static void main(String[] args) throws ClassNotFoundException {
        // new 创建对象
        //Test test = new Test();

        //读取静态字段
        //System.out.println(Test.NUM);

        //设置静态字段
        //Test.NUM = 2;

        //调用静态方法
        //Test.getNum();

        //反射调用
        //Class.forName("com.buydeem.reflect.Test",false,App1.class.getClassLoader());//不会
        Class.forName("com.buydeem.reflect.Test",true,App1.class.getClassLoader());//会
    }
}

class Test extends TestParent{
    public static Integer NUM = 1;

    static {
        System.out.println("执行静态代码块");
    }

    public static Integer getNum() {
        return NUM;
    }
}

class TestParent{
    static {
        System.out.println("父类执行静态代码块");
    }
}
```

对于其他方式的引用并不会触发初始化,例如下面的几种被动初始化方式:  

```java
public class App1 {
    public static void main(String[] args) throws ClassNotFoundException {
        //通过子类调用父类的静态变量
        System.out.println(Test.PARENT_NAME);
        //通过数组来定义类
        Test[] tests = new Test[10];
        //引用常量
        System.out.println(Test.NAME);
    }
}

class Test extends TestParent{
    public static final String NAME = "name";
    public static Integer NUM = 1;

    static {
        System.out.println("执行静态代码块");
    }

    public static Integer getNum() {
        return NUM;
    }
}

class TestParent{
    public static String PARENT_NAME;
    static {
        System.out.println("父类执行静态代码块");
    }
}
```

## 说明

该文为《深入理解Java虚拟机：JVM高级特性与最佳实践(第2版)》中第7章7.3类加载的过程所述,对书中内容作了大量删减,只留下主要的功能介绍.如果想查看详细介绍,可以看书中所述.
