# 消息发布订阅实现

guava中的EventBus在项目开发中,可以快速实现发布订阅模型,不需要我们自己去实现.下面记录一下如果使用

## EventBus使用

首先是创建EventBus,主要代码如下:

``` java
        //创建EventBus
        EventBus eventBus = new EventBus();
        //注册监听器
        eventBus.register(new MessageLister());
        //发送消息
        eventBus.post(new ParentMessage("1","this is parent message"));
        eventBus.post(new ChildMessage("2","this is child message","send"));
        eventBus.post("this is dead message");
```

上面的代码主要第一步是创建EventBus.接着是注册监听器,然后是发送消息.代码中的注册监听器就是创建一个订阅消息的对象,主要代码如下:

```java
public class MessageLister {

    /**
     * 处理类型为 ParentMessage消息
     * @param message
     */
    @Subscribe
    public void processParentMessage(ParentMessage message){
        String name = Thread.currentThread().getName();
        System.out.println(String.format("当前线程名称:[%s],处理器名称:processParentMessage,消息处理器名称:[%s]",name,message.toString()));
    }

    /**
     * 处理类型为 ChildMessage消息
     * @param message
     */
    @Subscribe
    public void processChildMessage(ChildMessage message){
        String name = Thread.currentThread().getName();
        System.out.println(String.format("当前线程名称:[%s],处理器名称:processChildMessage,消息处理器名称:[%s]",name,message.toString()));
    }

    /**
     * 处理死信消息
     * @param event
     */
    @Subscribe
    public void processDeadEventMessage(DeadEvent event){
        String name = Thread.currentThread().getName();
        System.out.println(String.format("当前线程名称:[%s],处理器名称:processDeadEventMessage,消息处理器名称:[%s]",name,event.getEvent().toString()));
    }
}
```

上面的注册了三个监听器,使用```@Subscribe```注解标记就行,方法的参数就是你要处理的消息类型.需要注意的点有一下几点:

- 注册成消息处理的方法只能有一个参数,并且方法必须是public

- 如果有一个消息类型为A,另外有一个消息类型为B,且A是B的父类,当你发送的消息类型为B时,处理A消息的方法和处理B消息的方法都会收到消息

- 如果你发送的消息没有找到对应的消息处理方法,那么会被死信消息处理方法处理.

最后运行上面代码的结果如下:

``` log
当前线程名称:[main],处理器名称:processParentMessage,消息处理器名称:[ParentMessage(id=1, content=this is parent message)]
当前线程名称:[main],处理器名称:processChildMessage,消息处理器名称:[ChildMessage(actionName=send)]
当前线程名称:[main],处理器名称:processParentMessage,消息处理器名称:[ChildMessage(actionName=send)]
当前线程名称:[main],处理器名称:processDeadEventMessage,消息处理器名称:[this is dead message]
```

## AsyncEventBus

与EventBus不同,AsyncEventBus是异步的.它的实现主要是在于提供了一个执行消息处理的线程池和消息的分发策略上.可以通过它的实现看出:

``` java
public class AsyncEventBus extends EventBus {

  
  public AsyncEventBus(String identifier, Executor executor) {
    super(identifier, executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }

  public AsyncEventBus(Executor executor, SubscriberExceptionHandler subscriberExceptionHandler) {
    super("default", executor, Dispatcher.legacyAsync(), subscriberExceptionHandler);
  }

  public AsyncEventBus(Executor executor) {
    super("default", executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }
```

其中```Dispatcher```有三种实现,分别为:

- ```mmediateDispatcher```：直接在当前线程中遍历所有的观察者并进行事件分发；

- ```LegacyAsyncDispatcher```：异步方法，存在两个循环，一先一后，前者用于不断往全局的队列中塞入封装的观察者对象，后者用于不断从队列中取出观察者对象进行事件分发；实际上，EventBus有个字类AsyncEventBus就是用该分发器进行事件分发的。

- ```PerThreadQueuedDispatcher```：这种分发器使用了两个线程局部变量进行控制，当dispatch()方法被调用的时候，会先获取当前线程的观察者队列，并将传入的观察者列表传入到该队列中；然后通过一个布尔类型的线程局部变量，判断当前线程是否正在进行分发操作，如果没有在进行分发操作，就通过遍历上述队列进行事件分发。

下面通过主要代码来看如何使用:

``` java
public static void main(String[] args) {
        //创建线程工厂
        ThreadFactory threadFactory = new ThreadFactory() {
            private static final String prefixName = "处理消息线程-";
            private int count = 1;
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName(prefixName+count);
                count++;
                return t;
            }
        };
        //创建线程池
        ThreadPoolExecutor executor = new ThreadPoolExecutor(4, 4, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(), threadFactory);
        //创建异步消息处理
        AsyncEventBus asyncEventBus = new AsyncEventBus(executor);
        asyncEventBus.register(new MessageLister());
        //发送消息
        for (int i = 0; i < 10; i++) {
            String message = "message id is :" + i;
            asyncEventBus.post(message);
            System.out.println("已发送消息 = "+message);
        }
    }
```

监听方法主要内容如下:

``` java
@Subscribe
    public void processMessage(Object event) throws InterruptedException {
        Random r = new Random();
        int number = r.nextInt(10);
        Thread.sleep(number*1000);
        String name = Thread.currentThread().getName();
        System.out.println(String.format("当前线程名称:[%s],消息内容为:[%s]",name,event.toString()));
    }
```

运行结果如下:

``` log
已发送消息 = message id is :0
已发送消息 = message id is :1
已发送消息 = message id is :2
已发送消息 = message id is :3
已发送消息 = message id is :4
已发送消息 = message id is :5
已发送消息 = message id is :6
已发送消息 = message id is :7
已发送消息 = message id is :8
已发送消息 = message id is :9
当前线程名称:[处理消息线程-1],消息内容为:[message id is :0]
当前线程名称:[处理消息线程-1],消息内容为:[message id is :4]
当前线程名称:[处理消息线程-3],消息内容为:[message id is :2]
当前线程名称:[处理消息线程-3],消息内容为:[message id is :6]
当前线程名称:[处理消息线程-3],消息内容为:[message id is :7]
当前线程名称:[处理消息线程-3],消息内容为:[message id is :8]
当前线程名称:[处理消息线程-3],消息内容为:[message id is :9]
当前线程名称:[处理消息线程-4],消息内容为:[message id is :3]
当前线程名称:[处理消息线程-2],消息内容为:[message id is :1]
当前线程名称:[处理消息线程-1],消息内容为:[message id is :5]
```

可以看出,线程是我们的```ThreadFactory```创建的,而且当消息还未处理完成时,还是可以继续发送消息.  
代码地址:[示例代码](https://github.com/pkq-zc/eventbus "示例代码")
