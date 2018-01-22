---
title: 徒手撸框架--实现Aop
date: 2018-01-13 11:12:48
tags: [java,aop,spring]
---

原文地址:[犀利豆的博客](https://www.xilidou.com/2018/01/13/spring-aop/)

上一讲我们讲解了Spring 的 IoC 实现。大家可以去我的博客查看[点击链接](https://www.xilidou.com/2018/01/08/spring-ioc/)，这一讲我们继续说说 Spring 的另外一个重要特性 AOP。之前在看过的大部分教程，对于Spring Aop的实现讲解的都不太透彻，大部分文章介绍了Spring Aop的底层技术使用了动态代理，至于Spring Aop的具体实现都语焉不详。这类文章看以后以后，我脑子里浮现的就是这样一个画面：

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnmxxxany9j30c80m0gm9.jpg)

我的想法就是，带领大家，首先梳理 Spring Aop的实现，然后屏蔽细节，自己实现一个Aop框架。加深对Spring Aop的理解。在了解上图1-4步骤的同时，补充 4 到 5 步骤之间的其他细节。

读完这篇文章你将会了解：

* Aop是什么？
* 为什么要使用Aop？
* Spirng 实现Aop的思路是什么
* 自己根据Spring 思想实现一个 Aop框架

<!--more-->

# Aop 是什么？
面向切面的程序设计（aspect-oriented programming，AOP）。通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。

# 为什么需要使用Aop？
面向切面编程，实际上就是通过预编译或者动态代理技术在不修改源代码的情况下给原来的程序统一添加功能的一种技术。我们看几个关键词，第一个是“动态代理技术”，这个就是Spring Aop实现底层技术。第二个“不修改源代码”，这个就是Aop最关键的地方，也就是我们平时所说的非入侵性。。第三个“添加功能”，不改变原有的源代码，为程序添加功能。

举个例子：如果某天你需要统计若干方法的执行时间，如果不是用Aop技术，你要做的就是为每一个方法开始的时候获取一个开始时间，在方法结束的时候获取结束时间。二者之差就是方法的执行时间。如果对每一个需要统计的方法都做如上的操作，那代码简直就是灾难。如果我们使用Aop技术，在不修改代码的情况下，添加一个统计方法执行时间的切面。代码就变得十分优雅。具体这个切面怎么实现？看完下面的文章你一定就会知道。

# Spring Aop 是怎么实现的？

所谓：
>计算机程序 = 数据结构 + 算法

在阅读过Spring源码之后，你就会对这个说法理解更深入了。

Spring Aop实现的代码非常非常的绕。也就是说 Spring 为了灵活做了非常深层次的抽象。同时 Spring为了兼容 `@AspectJ` 的Aop协议，使用了很多 Adapter （适配器）模式又进一步的增加了代码的复杂程度。
Spring 的 Aop 实现主要以下几个步骤：

1. 初始化 Aop 容器。
2. 读取配置文件。
3. 将配置文件装换为 Aop 能够识别的数据结构 -- `Advisor`。这里展开讲一讲这个advisor。Advisor对象中包又含了两个重要的数据结构，一个是 `Advice`，一个是 `Pointcut`。`Advice`的作用就是描述一个切面的行为，`pointcut`描述的是切面的位置。两个数据结的组合就是”在哪里，干什么“。这样 `Advisor` 就包含了”在哪里干什么“的信息，就能够全面的描述切面了。
4. Spring 将这个 Advisor 转换成自己能够识别的数据结构 -- `AdvicedSupport`。Spirng 动态的将这些方法拦截器织入到对应的方法。
5. 生成动态代理代理。
6. 提供调用，在使用的时候，调用方调用的就是代理方法。也就是已经织入了增强方法的方法。

# 自己实现一个 Aop 框架

同样，我也是参考了Aop的设计。只实现了基于方法的拦截器。去除了很多的实现细节。

使用上一讲的 IoC 框架管理对象。使用 Cglib 作为动态代理的基础类。使用 maven 管理 jar 包和 module。所以上一讲的 IoC 框架会作为一个 modules 引入项目。

下面我们就来实现我们的Aop 框架吧。

首先来看看代码的基本结构。

![](https://ws4.sinaimg.cn/large/006tNc79ly1fnombbcrkhj30c00heabs.jpg)

代码结构比上一讲的 IoC 复杂不少。我们首先对包每个包都干了什么做一个简单介绍。

* `invocation` 描述的就是一个方法的调用。注意这里指的是“方法的调用”，而不是调用这个动作。
* `interceptor` 大家最熟悉的拦截器，拦截器拦截的目标就是 `invcation` 包里面的调用。
* `advisor` 这个包里的对象，都是用来描述切面的数据结构。
* `adapter` 这个包里面是一些适配器方法。对于"适配器"不了解的同学可以去看看"设计模式"里面的"适配模式"。他的作用就是将 `advice` 包里的对象适配为 `interceptor`。
* `bean` 描述我们 json 配置文件的对象。
* `core` 我们框架的核心逻辑。

这个时候宏观的看我们大概梳理出了一条路线， `adaper` 将 `advisor` 适配为 `interceptor` 去拦截 `invoction`。

下面我们从这个链条的最末端讲起：

### `invcation`

首先 `MethodInvocation` 作为所有方法调用的接口。要描述一个方法的调用包含三个方法，获取方法本身`getMethod`,获取方法的参数`getArguments`，还有执行方法本身`proceed()`。

```java
public interface MethodInvocation {
    Method getMethod();
    Object[] getArguments();
    Object proceed() throws Throwable;
}
```

`ProxyMethodInvocation` 看名字就知道，是代理方法的调用，增加了一个获取代理的方法。
```java
public interface ProxyMethodInvocation extends MethodInvocation {
    Object getProxy();
}
```

### `interceptor`

`AopMethodInterceptor` 是 Aop 容器所有拦截器都要实现的接口：
```java
public interface AopMethodInterceptor {
    Object invoke(MethodInvocation mi) throws Throwable;
}
```

同时我们实现了两种拦截器`BeforeMethodAdviceInterceptor`和`AfterRunningAdviceInterceptor`,顾名思义前者就是在方法执行以前拦截，后者就在方法运行结束以后拦截：
```java
public class BeforeMethodAdviceInterceptor implements AopMethodInterceptor {
    private BeforeMethodAdvice advice;
    public BeforeMethodAdviceInterceptor(BeforeMethodAdvice advice) {
        this.advice = advice;
    }
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        advice.before(mi.getMethod(),mi.getArguments(),mi);
        return mi.proceed();
    }
}
```

```java
public class AfterRunningAdviceInterceptor implements AopMethodInterceptor {
    private AfterRunningAdvice advice;

    public AfterRunningAdviceInterceptor(AfterRunningAdvice advice) {
        this.advice = advice;
    }

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        Object returnVal = mi.proceed();
        advice.after(returnVal,mi.getMethod(),mi.getArguments(),mi);
        return returnVal;
    }
}

```

看了上面的代码我们发现，实际上 `mi.proceed()`才是执行原有的方法。而`advice`我们上文就说过，是描述增强的方法”干什么“的数据结构，所以对于这个before拦截器，我们就把advice对应的增强方法放在了真正执行的方法前面。而对于after拦截器而言，就放在了真正执行的方法后面。

这个时候我们过头来看最关键的 `ReflectioveMethodeInvocation`
```java
public class ReflectioveMethodeInvocation implements ProxyMethodInvocation {
    public ReflectioveMethodeInvocation(Object proxy, Object target, Method method, Object[] arguments, List<AopMethodInterceptor> interceptorList) {
        this.proxy = proxy;
        this.target = target;
        this.method = method;
        this.arguments = arguments;
        this.interceptorList = interceptorList;
    }

    protected final Object proxy;

    protected final Object target;

    protected final Method method;

    protected Object[] arguments = new Object[0];

    //存储所有的拦截器
    protected final List<AopMethodInterceptor> interceptorList;

    private int currentInterceptorIndex = -1;

    @Override
    public Object getProxy() {
        return proxy;
    }

    @Override
    public Method getMethod() {
        return method;
    }

    @Override
    public Object[] getArguments() {
        return arguments;
    }

    @Override
    public Object proceed() throws Throwable {

        //执行完所有的拦截器后，执行目标方法
        if(currentInterceptorIndex == this.interceptorList.size() - 1) {
            return invokeOriginal();
        }

        //迭代的执行拦截器。回顾上面的讲解，我们实现的拦击都会执行 im.proceed() 实际上又会调用这个方法。实现了一个递归的调用，直到执行完所有的拦截器。
        AopMethodInterceptor interceptor = interceptorList.get(++currentInterceptorIndex);
        return interceptor.invoke(this);

    }

    protected Object invokeOriginal() throws Throwable{
        return ReflectionUtils.invokeMethodUseReflection(target,method,arguments);
    }

}
```

在实际的运用中，我们的方法很可能被多个方法的拦截器所增强。所以我们，使用了一个list来保存所有的拦截器。所以我们需要递归的去增加拦截器。当处理完了所有的拦截器之后，才会真正调用调用被增强的方法。我们可以认为，前文所述的动态的织入代码就发生在这里。

```java
public class CglibMethodInvocation extends ReflectioveMethodeInvocation {

    private MethodProxy methodProxy;

    public CglibMethodInvocation(Object proxy, Object target, Method method, Object[] arguments, List<AopMethodInterceptor> interceptorList, MethodProxy methodProxy) {
        super(proxy, target, method, arguments, interceptorList);
        this.methodProxy = methodProxy;
    }

    @Override
    protected Object invokeOriginal() throws Throwable {
        return methodProxy.invoke(target,arguments);
    }
}
```
`CglibMethodInvocation` 只是重写了 `invokeOriginal` 方法。使用代理类来调用被增强的方法。

### advisor

这个包里面都是一些描述切面的数据结构，我们讲解两个重要的。
```java
@Data
public class Advisor {
    //干什么
    private Advice advice;
    //在哪里
    private Pointcut pointcut;

}
```
如上文所说，advisor 描述了在哪里，干什么。
```java
@Data
public class AdvisedSupport extends Advisor {
    //目标对象
    private TargetSource targetSource;
    //拦截器列表
    private List<AopMethodInterceptor> list = new LinkedList<>();

    public void addAopMethodInterceptor(AopMethodInterceptor interceptor){
        list.add(interceptor);
    }

    public void addAopMethodInterceptors(List<AopMethodInterceptor> interceptors){
        list.addAll(interceptors);
    }

}
```
这个`AdvisedSupport`就是 我们Aop框架能够理解的数据结构，这个时候问题就变成了--对于哪个目标，增加哪些拦截器。

### `core`
有了上面的准备，我们就开始讲解核心逻辑了。
```java
@Data
public class CglibAopProxy implements AopProxy{
    private AdvisedSupport advised;
    private Object[] constructorArgs;
    private Class<?>[] constructorArgTypes;
    public CglibAopProxy(AdvisedSupport config){
        this.advised = config;
    }

    @Override
    public Object getProxy() {
        return getProxy(null);
    }
    @Override
    public Object getProxy(ClassLoader classLoader) {
        Class<?> rootClass = advised.getTargetSource().getTagetClass();
        if(classLoader == null){
            classLoader = ClassUtils.getDefultClassLoader();
        }
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(rootClass.getSuperclass());
        //增加拦截器的核心方法
        Callback callbacks = getCallBack(advised);
        enhancer.setCallback(callbacks);
        enhancer.setClassLoader(classLoader);
        if(constructorArgs != null && constructorArgs.length > 0){
            return enhancer.create(constructorArgTypes,constructorArgs);
        }
        return enhancer.create();
    }
    private Callback getCallBack(AdvisedSupport advised) {
        return new DynamicAdvisedIcnterceptor(advised.getList(),advised.getTargetSource());
    }
}
```
`CglibAopProxy`就是我们代理对象生成的核心方法。使用 cglib 生成代理类。我们可以与之前ioc框架的代码。比较发现区别就在于：
```java
    Callback callbacks = getCallBack(advised);
    enhancer.setCallback(callbacks);
```
callback与之前不同了，而是写了一个`getCallback()`的方法，我们就来看看 getCallback 里面的 `DynamicAdvisedIcnterceptor`到底干了啥。

篇幅问题，这里不会介绍 cglib 的使用，对于callback的作用，不理解的同学需要自行学习。

```java
public class DynamicAdvisedInterceptor implements MethodInterceptor{

    protected final List<AopMethodInterceptor> interceptorList;
    protected final TargetSource targetSource;

    public DynamicAdvisedInterceptor(List<AopMethodInterceptor> interceptorList, TargetSource targetSource) {
        this.interceptorList = interceptorList;
        this.targetSource = targetSource;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        MethodInvocation invocation = new CglibMethodInvocation(obj,targetSource.getTagetObject(),method, args,interceptorList,proxy);
        return invocation.proceed();
    }
}
``` 

这里需要注意，`DynamicAdvisedInterceptor`这个类实现的 MethodInterceptor 是 gclib的接口，并非我们之前的 AopMethodInterceptor。

我们近距离观察 intercept 这个方法我们看到：
```java
MethodInvocation invocation = new CglibMethodInvocation(obj,targetSource.getTagetObject(),method, args,interceptorList,proxy);
```
通过这行代码，我们的整个逻辑终于连起来了。也就是这个动态的拦截器，把我们通过 `CglibMethodInvocation` 织入了增强代码的方法，委托给了 cglib 来生成代理对象。

至此我们的 Aop 的核心功能就实现了。

### AopBeanFactoryImpl

```java
public class AopBeanFactoryImpl extends BeanFactoryImpl{

    private static final ConcurrentHashMap<String,AopBeanDefinition> aopBeanDefinitionMap = new ConcurrentHashMap<>();

    private static final ConcurrentHashMap<String,Object> aopBeanMap = new ConcurrentHashMap<>();

    @Override
    public Object getBean(String name) throws Exception {
        Object aopBean = aopBeanMap.get(name);

        if(aopBean != null){
            return aopBean;
        }
        if(aopBeanDefinitionMap.containsKey(name)){
            AopBeanDefinition aopBeanDefinition = aopBeanDefinitionMap.get(name);
            AdvisedSupport advisedSupport = getAdvisedSupport(aopBeanDefinition);
            aopBean = new CglibAopProxy(advisedSupport).getProxy();
            aopBeanMap.put(name,aopBean);
            return aopBean;
        }

        return super.getBean(name);
    }
    protected void registerBean(String name, AopBeanDefinition aopBeanDefinition){
        aopBeanDefinitionMap.put(name,aopBeanDefinition);
    }

    private AdvisedSupport getAdvisedSupport(AopBeanDefinition aopBeanDefinition) throws Exception {

        AdvisedSupport advisedSupport = new AdvisedSupport();
        List<String> interceptorNames = aopBeanDefinition.getInterceptorNames();
        if(interceptorNames != null && !interceptorNames.isEmpty()){
            for (String interceptorName : interceptorNames) {

                Advice advice = (Advice) getBean(interceptorName);

                Advisor advisor = new Advisor();
                advisor.setAdvice(advice);

                if(advice instanceof BeforeMethodAdvice){
                    AopMethodInterceptor interceptor = BeforeMethodAdviceAdapter.getInstants().getInterceptor(advisor);
                    advisedSupport.addAopMethodInterceptor(interceptor);
                }

                if(advice instanceof AfterRunningAdvice){
                    AopMethodInterceptor interceptor = AfterRunningAdviceAdapter.getInstants().getInterceptor(advisor);
                    advisedSupport.addAopMethodInterceptor(interceptor);
                }

            }
        }

        TargetSource targetSource = new TargetSource();
        Object object = getBean(aopBeanDefinition.getTarget());
        targetSource.setTagetClass(object.getClass());
        targetSource.setTagetObject(object);
        advisedSupport.setTargetSource(targetSource);
        return advisedSupport;

    }

}

```
`AopBeanFactoryImpl`是我们产生代理对象的工厂类，继承了上一讲我们实现的 IoC 容器的BeanFactoryImpl。重写了 getBean方法，如果是一个切面代理类，我们使用Aop框架生成代理类，如果是普通的对象，我们就用原来的IoC容器进行依赖注入。
`getAdvisedSupport`就是获取 Aop 框架认识的数据结构。

剩下没有讲到的类都比较简单，大家看源码就行。与核心逻辑无关。

## 写个方法测试一下

我们需要统计一个方法的执行时间。面对这个需求我们怎么做？
```java
public class StartTimeBeforeMethod implements BeforeMethodAdvice{
    @Override
    public void before(Method method, Object[] args, Object target) {
        long startTime = System.currentTimeMillis();
        System.out.println("开始计时");
        ThreadLocalUtils.set(startTime);
    }
}
```

```java
public class EndTimeAfterMethod implements AfterRunningAdvice {
    @Override
    public Object after(Object returnVal, Method method, Object[] args, Object target) {
        long endTime = System.currentTimeMillis();
        long startTime = ThreadLocalUtils.get();
        ThreadLocalUtils.remove();
        System.out.println("方法耗时：" + (endTime - startTime) + "ms");
        return returnVal;
    }
}
```
方法开始前，记录时间，保存到 ThredLocal里面，方法结束记录时间，打印时间差。完成统计。

目标类：
```java
public class TestService {
    public void testMethod() throws InterruptedException {
        System.out.println("this is a test method");
        Thread.sleep(1000);
    }
}
```

配置文件：
```json
[
  {
    "name":"beforeMethod",
    "className":"com.xilidou.framework.aop.test.StartTimeBeforeMethod"
  },
  {
    "name":"afterMethod",
    "className":"com.xilidou.framework.aop.test.EndTimeAfterMethod"
  },
  {
    "name":"testService",
    "className":"com.xilidou.framework.aop.test.TestService"
  },
  {
    "name":"testServiceProxy",
    "className":"com.xilidou.framework.aop.core.ProxyFactoryBean",
    "target":"testService",
    "interceptorNames":[
      "beforeMethod",
      "afterMethod"
    ]
  }
]
```
测试类：

```java
public class MainTest {
    public static void main(String[] args) throws Exception {
        AopApplictionContext aopApplictionContext = new AopApplictionContext("application.json");
        aopApplictionContext.init();
        TestService testService = (TestService) aopApplictionContext.getBean("testServiceProxy");
        testService.testMethod();
    }
}
```

最终我们的执行结果：
``` shell
开始计时
this is a test method
方法耗时：1015ms

Process finished with exit code 0
```

至此 Aop 框架完成。

# 后记

Spring 的两大核心特性 IoC 与 Aop 两大特性就讲解完了，希望大家通过我写的两篇文章能够深入理解两个特性。

Spring的源码实在是复杂，阅读起来常常给人极大的挫败感，但是只要能够坚持，并采用一些行之有效的方法。还是能够理解Spring的代码。并且从中汲取营养。

下一篇文章，我会给大家讲讲阅读开源代码的一些方法和我自己的体会，敬请期待。

## 最后 

github：https://github.com/diaozxin007/framework



