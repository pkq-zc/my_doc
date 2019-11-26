# spring中Aware相关接口的作用

## 简介

在spring中有很多以```Aware```结尾的接口.如果一个Bean实现了该接口,那么当该Bean被spring初始化时,spring会向该Bean注入相关资源(就是会回调接口中的方法).

## 示例说明

``` java
@Component
public class TestService implements BeanNameAware,ApplicationContextAware {
    private String beanName;
    private ApplicationContext context;
    @Override
    public void setBeanName(String name) {
        System.out.println("name = " + name);
        beanName = name;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("applicationContext = " + applicationContext);
        context = applicationContext;
    }
}
```

上面的```TestService```实现了两个接口```BeanNameAware```和```ApplicationContextAware```.当spring对bean进行初始化时,spring会调用接口对应的方法.这样我们就可以获取到spring中的资源(本示例中的beanName和context).

## 常用的Aware相关接口作用说明

|接口名称|作用|
|--|--|
|ApplicationContextAware|获取spring 上下文环境的对象|
|BeanNameAware|获取该bean在BeanFactory配置中的名字|
|BeanFactoryAware|创建它的BeanFactory实例|
|ServletContextAware|获取servletContext容器|
|ResourceLoaderAware|获取ResourceLoader对象,通过它获得各种资源|
