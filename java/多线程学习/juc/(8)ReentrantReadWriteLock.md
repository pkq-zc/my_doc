# ReentrantReadWriteLock

```ReentrantReadWriteLock```是一个读写锁,与```ReentrantLock```不同之处在于它提供两种模式的锁,一种为读锁另一种为写锁.读锁是一种共享锁,即当一个线程已经拥有了读锁,其他线程仍然可以继续请求获取读锁,但是不能获取写锁.写锁则是一种排他锁,即当一个线程已经拥有了写锁,那么其他线程就无法获取读锁或者写锁.  
例如我们应用中的缓存,允许多个线程同时读取数据,但是对于写入数据的情况就只能允许一个线程进行.下面我们使用代码简单的实现一个缓存.  

```java
public class App9 {
    public static void main(String[] args) throws InterruptedException {
        Cache cache = new Cache();
        //更新缓存线程
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+":写入缓存");
                cache.put("info","测试内容");
            }
        });
        t1.setName("t1");
        t1.start();

        Thread.sleep(500);

        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                String info = cache.get("info");
                System.out.println(Thread.currentThread().getName() + ":" + info);
            }
        };

        Thread t2 = new Thread(runnable);
        t2.setName("t2");
        t2.start();

        Thread t3 = new Thread(runnable);
        t3.setName("t3");
        t3.start();

    }
}

class Cache{
    //读写锁
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    //存储缓存数据
    private Map<String,String> map = new HashMap<>();

    /**
     * 存入数据
     * @param key
     * @param value
     */
    public void put(String key,String value){
        //添加写锁
        lock.writeLock().lock();
        //放入数据
        map.put(key,value);
        //2s之后再释放锁
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //解锁
        lock.writeLock().unlock();
    }

    /**
     * 获取数据
     * @param key
     * @return
     */
    public String get(String key){
        //获取读锁
        lock.readLock().lock();
        //获取数据
        String value = map.get(key);
        //2s之后再释放锁
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.readLock().unlock();
        return value;
    }
}
```

执行结果如下:

```java
t1:写入缓存
t2:测试内容
t3:测试内容
```

上面的代码中,我们让写入缓存线程t1先获取锁,而读取缓存的线程t2和她t3等待写锁释放.当写锁释放之后,读线程t2和t3立马都打印出缓存数据.由此可以说明,写锁再被当前线程获取之后,其他线程都无法再次获取到读锁或者写锁.而读锁被获取后,其他线程是可以获取到写锁的.  
除此之外,```ReentrantReadWriteLock```还有以下几个特征:

- ```可重入性```:这个跟```ReentrantLock```一样也支持重入.即当前线程已获得锁,接下来它可以再次获取锁.
- ```锁降级```:它在支持重入的情况下,还支持从写锁降级到读锁.但是需要注意的是读锁是无法降级成读锁的.

```java
public class App10 {
    public static void main(String[] args) {
        ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                //获取写锁
                lock.writeLock().lock();
                System.out.println("获取写锁");
                //获取读锁
                lock.readLock().lock();
                System.out.println("获取读锁");
                //释放读锁
                lock.readLock().unlock();
                System.out.println("释放读锁");
                //释放写锁
                lock.writeLock().unlock();
                System.out.println("释放写锁");
            }
        });
        t1.setName("t1");
        t1.start();
    }
}
```

上面就是锁降级的例子.在上面的例子中,线程先获取```写锁```,然后再获取```读锁```.最后程序能顺利打印出结果并结束.如果先获取```读锁```再获取```写锁```,这时候线程在获取写锁的时候会阻塞永远无法获取到```写锁```.  

同样读写锁也支持```Condition```,读写锁除了锁区分读锁和写锁之外,其他功能跟```ReentrantLock```基本上就类似了.如果还不了解```ReentrantLock```如何使用,可以看我之前写的相关[文章](https://www.jianshu.com/p/3f00c9588c32).
