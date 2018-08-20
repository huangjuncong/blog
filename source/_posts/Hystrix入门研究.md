---
title: Hystrix入门研究
date: 2017-10-23 23:59:28
tags: [后端,Hystrix,java]
---
## 1、Hystrix是什么
Hystrix 是Netflix开源的一个针对分布式容错和库。Hystrix的主要功能是隔离分布式系统之间的故障，防止故障带来的雪崩效应。同时也能提供一个分布式服务的优雅的降级方案。从而提高系统的可用性的组件。


<!--more-->


## 2、Hystrix设计理念是什么（其实也是高可用系统设计的理念）？
1. 防止单个系统故障后，造成容器（tomcat，scf）的线程全部占满，影响服务响应。
2. 使用快速失败和泄洪代替队列等待。
3. 在系统故障之后提供优雅的降级措施。
4. 使用隔离技术降低故障影响面。
5. 提供准实时的监控报警系统。
6. 提供准实时动态的配置系统。å
7. 客户端感知下游服务状态，防止错误的发展，而不通过真实的调用就能感知。

## 3、Hystrix怎么用
* Hello World：

```java
public class CommandHelloWorld extends HystrixCommand<String>{
 
    private final String name;
 
    public CommandHelloWorld(String name){
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }
 
    protected String run() throws Exception {
        return "Hello " + name;
    }
  
    public static void main(String[] args) throws ExecutionException, InterruptedException {
    String s = new CommandHelloWorld("BoB").execute();
 
    System.out.println(s);
 
    }
}
```

* 降级：

```java
public class CommandHelloFailure extends HystrixCommand<String> {
 
    private final String name;
 
    public CommandHelloFailure(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }
 
    @Override
    protected String run() {
        throw new RuntimeException("this command always fails");
    }
 
    @Override
    protected String getFallback() {
        return "Hello Failure " + name + "!";
    }
}
``` 


## 4、Hystrix实现思路分析
### 1、数据流

![数据流](http://upload-images.jianshu.io/upload_images/4268675-43d60574a43c0bdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 初始化 `HystrixCommand` 或者 `HystrixObservableCommand` 对象。
2. 执行。
3. 判断是否有缓存？
4. 判断是否调用链路是否通畅？
5. 判断线程池/队列/信号量 是否满了？
6. 执行`HystrixObservableCommand.construct()` 或者`HystrixCommand.run()`方法
7. 计算调用下游的健康程度
8. 判断时候需要降级
9. 完成请求

### 2、熔断器
![](http://upload-images.jianshu.io/upload_images/4268675-bfcad103617fc7b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
每个熔断器维护10个buckets窗口，每秒生成一个新的bucket，把最早的bucket抛弃，每个bucket记录了调用的，成功、失败、超时、拒绝的次数，如果失败数量达到某个阈值，就会触发熔断。
### 3、隔离
#### 线程隔离
每个下游调用使用独立的线程池，而非与请求的调用共用一个线程池，这样可以防止失败的调用占用共用的线程池，造成整个系统拒绝服务。
#### 优缺点
优点：
相互独立，减少互相影响的风险，总的来说就是隔离解耦，不会互相影响》
缺点：
过多的线程池造成cpu计算能力的消耗，和增加代码的复杂度。
#### 信号量隔离   
信号隔离也可以用于限制并发访问，防止阻塞扩散, 与线程隔离最大不同在于执行依赖代码的线程依然是请求线程（该线程需要通过信号申请）,
   如果客户端是可信的且可以快速返回，可以使用信号隔离替换线程隔离,降低开销.
   
### 4、请求折叠
可以使用组件`HystrixCollapser`把前端的多个请求折叠为单一的一个后端请求。减少线程和链接的开销。
### 5、请求缓存
把请求缓存起来。这个不过多解释了
