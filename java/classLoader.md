# java之ClassLoader

## 1.作用

主要作用为将编译的class加载到JVM,同时确定每个类应该由哪个类加载器加载.

## 2.类型

java默认主要提供了三个ClassLoader,分别为以下三个:

- ```BootStrap ClassLoader```启动类加载器.它主要负责加载Java核心类库```$JAVA_HOME/jre/lib```.

- ```ExtClassLoader```扩展类加载器.它主要负责加载扩展类库```$JAVA_HOME/jre/lib/ext```和系统指定目录```System.getProperty("java.ext.dirs")```.

- ```AppClassLoader```系统类加载器.它是我们平常使用最多的类加载器.它主要加载classpath目录下的所有jar和class.

示例代码:

```java
        ClassLoader bootstrapClassLoader = String.class.getClassLoader();
        System.out.println("bootstrapClassLoader = " + bootstrapClassLoader);
        ClassLoader extClassLoader = DNSNameService.class.getClassLoader();
        System.out.println("extClassLoader = " + extClassLoader);
        //APP类为自己定义的一个类
        ClassLoader appClassLoader = App.class.getClassLoader();
        System.out.println("appClassLoader = " + appClassLoader);
```

打印结果如下:

```
bootstrapClassLoader = null
extClassLoader = sun.misc.Launcher$ExtClassLoader@6d6f6e28
appClassLoader = sun.misc.Launcher$AppClassLoader@18b4aac2
```

输出的第一个为null是因为这是```BootStrap Classloader```由JVM实现,所以打印的为null.

## 3.双亲委派模型

![双亲委派模型](https://i.loli.net/2020/06/17/ZhBaHl5DTuzQGgw.png)  
当一个类加载器接受到请求时,首先会请求其父类加载器加载,每一层都是如此.当父类加载器无法找到该类时,子类才会去尝试加载该类.从下图可以看出类加载器的基本结构.
![类加载器基本结构](https://i.loli.net/2020/06/17/uSdzRp14KyJ6DaE.png)  
在```ClassLoader```类中可以找到此段代码,解释了双亲委派的流程:

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            //第一步检查类是否已经被加载了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    //如果父 类加载器不为空
                    if (parent != null) {
                        //调用父 类加载器
                        c = parent.loadClass(name, false);
                    } else {
                        //如果为空,则使用bootStrap ClassLoader尝试加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    //如果一直没找到,则按顺序调用findClass
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

## 为什么要使用双亲委派模型

试想如果没有双亲委派模型,我们可以自己去写一个类```java.lang.String```,然后使用自定义类加载器去加载这个类,那么JVM已经通过BootStrap ClassLoader加载了```java.lang.String```,我们自己也加载了一个```java.lang.String```类.这时候JVM中就有了两个类名一样的```java.lang.String```类,我们就无法通过类全限定名找到一个唯一的```java.lang.String```类了.总结来说其一出于安全角度,其二也避免了类的重复加载.

## 实现自己的类加载器

要实现自定义的类加载器主要是继承```ClassLoader```类,然后重写```findClass```方法.该方法在```ClassLoader```中的定义如下:

```java
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```

我们实现一个自定义的类加载器```MyClassLoader```.简要代码如下:

```java
public class MyClassLoader extends ClassLoader{
    private static final String PATH = "C:\\Users\\zengchao\\Desktop\\";

    public MyClassLoader(ClassLoader parent) {
        super(parent);
    }

    public MyClassLoader() {
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        int index = name.lastIndexOf(".");
        String className = index == -1 ? name : name.substring(index + 1, name.length()) + ".class";
        File file = new File(PATH + className);
        FileInputStream in;
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        try {
            in = new FileInputStream(file);
            byte[] buffer = new byte[1024];
            int num = 0;
            while ((num = in.read(buffer)) != -1){
                out.write(buffer,0,num);
            }
            byte[] classData = out.toByteArray();
            return defineClass(name,classData,0,classData.length);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

该类加载器会加载```PATH```目录下的```.class```文件.测试代码如下:

```java
public class App {
    public static void main(String[] args) throws ClassNotFoundException {
        MyClassLoader classLoader = new MyClassLoader();
        Class<?> clazz = classLoader.loadClass("com.buydeem.Student");
        System.out.println("clazz = " + clazz);
        System.out.println("clazz.getClassLoader() = " + clazz.getClassLoader());
    }
}
```

打印结果如下:

```
clazz = class com.buydeem.Student
clazz.getClassLoader() = sun.misc.Launcher$AppClassLoader@18b4aac2
```

如果没有理解双亲委派会觉得这个结果很奇怪.为什么会是```AppClassLoader```而不是我们自定义的```MyClassLoader```.因为双亲委派模型,它会通过先将类加载交给他的父类加载器,即先交给```AppClassLoader```,再交给```ExtClassLoader```,再交给```BootStrapClassLoader```.先前说过这几个类加载器加载的路径.我们编写的类可能不在```BootStrapClassLoader```和```ExtClassLoader```加载的目录下.所以这两个类加载器是无法加载我们定义的```Student```类.那为什么```APPClassLoader```可以加载```Student```类呢?可能我们平常没有注意使用编辑器启动代码前面打印的一大段启动参数.如果你将参数仔细查找一番,你就能发现其中的端倪:
![启动参数](https://i.loli.net/2020/06/23/wQEOtLpR2FIYG1j.png)
可以看出绿色框中指定了我们的编辑器编译好的class文件所在目录.
![编辑器target目录](https://i.loli.net/2020/06/23/vjPHTb9tM2rFclD.png)
我们可以将```target```中编译好的```Student.class```文件删除.这样```AppClassLoader```就无法找到```Student.class```文件.这样我们便可以使用自己自定义类加载器加载类了.
![删除target中的Student.class](https://i.loli.net/2020/06/23/QfAYLMWV9lrJazs.png)

## 如何打破双亲委派模型

我们并不一定要遵守双亲委派这个原则,我们同样也可以打破双亲委派模型.主要的方法有:

- 设置自定义的类加载器的父类加载器为空
- 重写```loadClass```方法

```java
public class App {
    public static void main(String[] args) throws ClassNotFoundException {
        MyClassLoader classLoader = new MyClassLoader(null);
        Class<?> clazz = classLoader.loadClass("com.buydeem.Student");
        System.out.println("clazz = " + clazz);
        System.out.println("clazz.getClassLoader() = " + clazz.getClassLoader());
    }
}
```

上面的代码即使不删除```target```中的```Student.class```文件,还是会使用我们自定义的```MyClassLoader```的加载类.

```java
public class MyClassLoader extends ClassLoader{
    private static final String PATH = "C:\\Users\\zengchao\\Desktop\\";

    public MyClassLoader(ClassLoader parent) {
        super(parent);
    }

    public MyClassLoader() {
    }

    // 重写loadClass方法,直接调用我们自己定义的findClass方法.打破双亲委派模型
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        System.out.println("name = " + name);
        return findClass(name);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        int index = name.lastIndexOf(".");
        String className = index == -1 ? name : name.substring(index + 1, name.length()) + ".class";
        File file = new File(PATH + className);
        FileInputStream in;
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        try {
            in = new FileInputStream(file);
            byte[] buffer = new byte[1024];
            int num = 0;
            while ((num = in.read(buffer)) != -1){
                out.write(buffer,0,num);
            }
            byte[] classData = out.toByteArray();
            return defineClass(name,classData,0,classData.length);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

上面的代码并不能正常执行,打印结果如下:

```
name = com.buydeem.Student
name = java.lang.Object
java.io.FileNotFoundException: C:\Users\zengchao\Desktop\Object.class (系统找不到指定的文件。)
	at java.io.FileInputStream.open0(Native Method)
	at java.io.FileInputStream.open(FileInputStream.java:195)
	at java.io.FileInputStream.<init>(FileInputStream.java:138)
	at com.buydeem.MyClassLoader.findClass(MyClassLoader.java:36)
	at com.buydeem.MyClassLoader.loadClass(MyClassLoader.java:25)
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at com.buydeem.MyClassLoader.findClass(MyClassLoader.java:43)
	at com.buydeem.MyClassLoader.loadClass(MyClassLoader.java:25)
	at com.buydeem.App.main(App.java:10)
Exception in thread "main" java.lang.NoClassDefFoundError: java/lang/Object
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at com.buydeem.MyClassLoader.findClass(MyClassLoader.java:43)
	at com.buydeem.MyClassLoader.loadClass(MyClassLoader.java:25)
	at com.buydeem.App.main(App.java:10)
```

上面报的异常是找不到```Object.class```文件,这是因为java中所有的类都隐式的继承了```Object```类.当加载```Student```类时,它还需要加载它的父类```Object```.而在我指定的目录中没有```Object.class```,所以会报找不到class文件.那我自己定义一个类```java.lang.Object```,然后编译完成放在自定义类的目录下可行吗?

```java
name = com.buydeem.Student
name = java.lang.Object
Exception in thread "main" java.lang.SecurityException: Prohibited package name: java.lang
	at java.lang.ClassLoader.preDefineClass(ClassLoader.java:662)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:761)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at com.buydeem.MyClassLoader.findClass(MyClassLoader.java:43)
	at com.buydeem.MyClassLoader.loadClass(MyClassLoader.java:25)
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at com.buydeem.MyClassLoader.findClass(MyClassLoader.java:43)
	at com.buydeem.MyClassLoader.loadClass(MyClassLoader.java:25)
	at com.buydeem.App.main(App.java:10)
```

再次尝试得上面的错误结果.即使我们打破了双亲委派机制,我们还是无法加载已```java.```开头的类.在```ClassLoader```中有这么一个方法:

```java
    private ProtectionDomain preDefineClass(String name,
                                            ProtectionDomain pd)
    {
        if (!checkName(name))
            throw new NoClassDefFoundError("IllegalName: " + name);

        // Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
        // relies on the fact that spoofing is impossible if a class has a name
        // of the form "java.*"
        if ((name != null) && name.startsWith("java.")) {
            throw new SecurityException
                ("Prohibited package name: " +
                 name.substring(0, name.lastIndexOf('.')));
        }
        if (pd == null) {
            pd = defaultDomain;
        }

        if (name != null) checkCerts(name, pd.getCodeSource());

        return pd;
    }
```

从源码中可以看出,以```java.*```是无法通过检查的.这样进一步保证了类加载的安全.那我们还是通过修改```loadClass```实现我们最初的想法.

```java
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        if (name.startsWith("java.")){
            return getSystemClassLoader().loadClass(name);
        }else {
            return findClass(name);
        }
    }
```

我们简单的修改了代码,使用系统类加载器加载我们无法加载的类,最后我们使用重写```loadClass```这种方打破了双亲委派机制.

## 类加载器对equals方法的影响

```java
public class App {
    public static void main(String[] args) throws ClassNotFoundException {
        MyClassLoader classLoader = new MyClassLoader();
        Class<?> clazz = classLoader.loadClass("com.buydeem.Student");
        System.out.println("clazz = " + clazz);
        System.out.println("clazz.getClassLoader() = " + clazz.getClassLoader());

        Class<?> clazz2 = ClassLoader.getSystemClassLoader().loadClass("com.buydeem.Student");
        Class<?> clazz3 = ClassLoader.getSystemClassLoader().loadClass("com.buydeem.Student");

        System.out.println("clazz2.equals(clazz) = " + clazz2.equals(clazz));
        System.out.println("clazz2.equals(clazz3) = " + clazz2.equals(clazz3));
    }
}
```

结果如下:

```
clazz = class com.buydeem.Student
clazz.getClassLoader() = com.buydeem.MyClassLoader@7382f612
clazz2.equals(clazz) = false
clazz2.equals(clazz3) = true
```

上面的结果clazz与clazz2不同是因为他们即使class文件一样,但是他们是通过不同的类加载器加载的,所以他们不不一样的.而通过同一个类加载器加载的同一个类,他们是相同的.