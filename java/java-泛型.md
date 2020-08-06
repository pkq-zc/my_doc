# java-泛型

## 什么是泛型

泛型的本质就是```类型参数化```.简单点说就是给类型指定一个参数,在使用的时候再指定参数的值.这样参数类型可以用在```类```,```接口```和```方法```中,分别叫做泛型类,泛型接口和泛型方法.

## 为什么使用泛型

泛型是在JDK1.5中才引入的,其最主要的目的就是将```类型检查提前到编译时期,将类型强转交给编译器完成```.对比下面连段代码你就知道了:

```java
  //未使用泛型
  @Test
  public void test1(){
    List list = new ArrayList();
    //无法限制加入的元素的类型
    list.add(1);
    list.add("hello");
    for (Object o : list) {
      //需要判断类型后才能强制转换
      if (o instanceof Integer){
        Integer i = (Integer) o;
        System.out.println("数字:"+i);
      }else {
        throw new RuntimeException("不是数字");
      }
    }
  }
  
  // 使用泛型
  @Test
  public void test2(){
    List<Integer> list = new ArrayList<>();
    //加入进去的元素必须是Integer
    list.add(1);
    list.add(2);
    for (Integer integer : list) {
      //使用时能直接获取到确定类型的元素
      System.out.println(integer);
    }
  }
```

上面List使用泛型和不使用泛型的区别一下子就看出来了.我们使用泛型后能确保我们放入的元素是同一种类型,且在获取时也不需要我们自己去判断能不能强转.因为使用泛型,所以上面的```类型检查和类型转换```的工作就交给编译器自己去解决了.

## 泛型的作用对象

泛型可以作用在类,接口和方法上,下面将一一介绍这三种情况.

### 泛型类

在类声明的时候指定参数就构成了泛型类.这么说你可能不太明白是什么意思,直接看代码.  

```java
class Generic<T>{
  private T data;

  public T getData() {
    return data;
  }

  public void setData(T data) {
    this.data = data;
  }

  @Override
  public String toString() {
    return "Generic{" +
      "data=" + data +
      '}';
  }
}
```

上面代码中的```T```就是我们指定的类型参数,```Generic<T>```就是泛型类,我们在该类的内部就可以使用```T```了.使用起来也很简单,代码如下:  

```java
public class GenericTest {
  @Test
  public void test1(){
    //声明T为String类型
    Generic<String> stringGeneric = new Generic<>();
    stringGeneric.setData("hello");
    System.out.println(stringGeneric);
    //声明T为Integer
    Generic<Integer> integerGeneric = new Generic<>();
    integerGeneric.setData(1);
    System.out.println(integerGeneric);
  }
}
```

### 泛型接口

定义跟使用与泛型类基本上相似,这里就不多说直接上例子.  

```java
interface TestInterface<T>{
   void sayHello(T content);
}
```

### 泛型方法

上面泛型类和泛型接口这个不容易搞混,但是下面的泛型方法容易搞混.定义我也说不明白直接看代码.

```java
class Generic<T>{
  private T data;
  //不是泛型方法
  public T getData() {
    return data;
  }
  //不是泛型方法
  public void setData(T data) {
    this.data = data;
  }
  //是泛型方法
  public <E> E sayHello(E data){
    return data;
  }

  @Override
  public String toString() {
    return "Generic{" +
      "data=" + data +
      '}';
  }
}
```

还是之前的泛型类,我们定义了一个方法方法```public <E> E sayHello(E data)```.这里面重点是在方法中声明了泛型参数```<E>```.如果没有声明泛型参数那就不是泛型方法.  

## 泛型类的继承和泛型接口的实现

如果我们在继承一个泛型类时,我们没有明确指定泛型类的类型,那么我们新创建的这个类也需要声明为泛型类.如果我们没有指定,那么默认指定类型为```Object```.  

```java
//指定泛型类的类型为String
class StringGeneric extends Generic<String>{
}
//默认为Object
class ObjectGeneric extends Generic{

}
//未指定
class ChildGeneric<T> extends Generic<T>{
}
```

泛型接口与泛型类差不多,这里面就不多说了.

## 泛型的上界和下界

泛型的上界使用```<? extends Number>```表示.意思是需要一个```Number```类型或者```Number```类型的子类型.但是需要注意的是```上界只能从其中获取数据而不能存放数据```.

```java
    //上边界 通常用在消费者模型
    Generic<? extends Number> numberGeneric = new Generic<>();
    Number data = numberGeneric.getData();
    //numberGeneric.setData(data);无法使用
```

泛型的下界使用```<? super Number>```表示.意思是需要一个```Number```类型或者```Number```类型的父类型.需要注意的是```下界只能从其中存数据而不能从放数据```.

```java
//下边界 通常用在生产者模型
    Generic<? super Number> numberGeneric2 = new Generic<>();
    numberGeneric2.setData(1);
    numberGeneric2.setData(100000L);
    numberGeneric2.setData(0.19);
```
