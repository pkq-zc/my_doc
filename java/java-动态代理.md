# java-动态代理

## １.什么是代理

在生活中这种例子非常常见.例如:我们去有一套闲置的房子需要出租,我们可以自己发出租消息,自己完成房屋出租.但是我们觉得太麻烦或者发布消息的渠道太少,自己处理比较麻烦,这个时候就可以找中介公司代理我们的业务.在程序中也一样,例如我们有一个保存用户信息的接口和其接口的实现.

```java
public interface UserDao {
    void save();
}

public class UserDaoImpl implements UserDao{
    @Override
    public void save() {
        System.out.println("保存用户");
    }
}
```

现在我们需要将保存用户信息的这个方法增强，让他可以支持事务.但是我们又不能修改```UserDaoImpl```中的实现.这个时候我们就可以用到代理.

## 2.静态代理

```java
public class StaticUserDao implements UserDao{
    private UserDao target;

    public StaticUserDao(UserDao target) {
        this.target = target;
    }

    @Override
    public void save() {
        System.out.println("开启事务");
        target.save();
        System.out.println("提交事务");
    }
}
```

我们在```StaticUserDao```持有对代理目标对象的引用，然后重写```save```方法,在原来的方法之前开启事务,在方法之后提交事务.调用代码如下:

```java
        //静态代理 写一个类实现接口，然后持有目标类的引用，重写目标方法
        System.out.println("======== 静态代理 ========");
        UserDaoImpl userDao = new UserDaoImpl();
        StaticUserDao staticUserDao = new StaticUserDao(userDao);
        staticUserDao.save();
```

结果如下:

```java
======== 静态代理 ========
开启事务
保存用户
提交事务
```

使用这种方法显然可以达到我们之前的要求,但是这种方法也有不好的地方.例如:```代码太过于冗余```,由于代理对象要实现与目标一致的接口,者将导致产生过多的代理类.```不易维护```,一旦接口新增方法,目标对象与代理类都要进行修改.

## 3.JDK动态代理

JDK动态代理,动态的在内存中建立代理对象,从而实现对目标对象的代理.

```java
        // jdk动态代理 需要目标类实现接口,代理接口上的方法来实现.如果被代理的方法不再接口上,无法实现
        System.out.println("========　jdk动态代理　========");
        // loader -> 代理类的类加载器
        // interfaces -> 代理类要实现的接口列表
        // h -> 代理处理逻辑
        UserDao jdkProxy = (UserDao)Proxy.newProxyInstance(UserDaoImpl.class.getClassLoader(),UserDaoImpl.class.getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("开启事务");
                method.invoke(userDao,args);
                System.out.println("提交事务");
                return null;
            }
        });
        System.out.println(jdkProxy.getClass().getName());
        jdkProxy.save();
```

结果如下所示:

```
========　jdk动态代理　========
com.sun.proxy.$Proxy0
开启事务
保存用户
提交事务
```

从结果可以看出,它完全可以实现我们的需求,同时它生成的代理类的类名是以```$```开头的代理类.它与静态代理不同的点在于,jdk动态代理在编译完成之后不会生成实际的class文件,它是在运行时才生成代理的class文件.但是使用jdk的动态代理也有一个限制,它需要目标类被代理的方法必须是接口上的方法才行.如果不是,则无法进行代理.

## 4.Cglib动态代理

CGLIB是一个功能强大,高性能的代码生成包.它为没有实现接口的类提供代理,为JDK的动态代理提供了很好的补充.因为不是原生java提供的,所以在使用之前需要先引入jar包.

```xml
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.3.0</version>
        </dependency>
```

使用Cglib动态代理实例代码如下:

```java
        System.out.println("========　Cglib动态代理　========");
        MethodInterceptor interceptor = new MethodInterceptor() {
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                System.out.println("开启事务");
                userDao.save();
                System.out.println("提交事务");
                return null;
            }
        };
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(userDao.getClass());
        enhancer.setCallback(interceptor);
        UserDao cglibUserDao = (UserDao) enhancer.create();
        System.out.println(cglibUserDao.getClass().getName());
        cglibUserDao.save();
```

运行结果如下:

```
========　Cglib动态代理　========
com.buydeem.UserDaoImpl$$EnhancerByCGLIB$$8f41b8d
开启事务
保存用户
提交事务
```

从结果很显然可以实现我们的需求.Cglib的原理就是动态生成一个要代理类的子类,子类重写要代理的类的所有不是```final```的方法.在子类中采用方法拦截的技术拦截所有父类方法的调用,织入横切逻辑.它比使用java反射的JDK动态代理要快.所以使用cglib需要被代理的类和方法不能是final修饰的.它的好处是没有接口上的限定.