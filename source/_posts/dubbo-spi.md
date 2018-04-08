---
title: dubbo-spi
date: 2018-04-07 14:13:20
tags:
---

在阅读dubbo的代码之前我们需要了解一下java 的 SPI 机制。SPI 全称 （Service Provider Interface）,实际上如果你了解了 Spirng 的 IoC 机制，那么 SPI 也具有异曲同工之妙。

为什么需要这个 SPI 机制，因为往往对于同一个模块我们有不同的技术实现，最常见的就比如，jdbc，日志系统，xml的解析等。面向对象的编程中，我们推荐使用面向接口的编程。模块之间基于接口编程，模块之间不适用硬编码来进行编写，降低模块之间的耦合。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。所以java就提供，为某个接口寻找服务实现的机制。

# SPI 的约定：
在jar包的 ClassPath 下面放置一个 META/service文件夹。

* 在文件命是要扩展的接口全名
* 文件内部是接口的实现类

使用：

```java

ServiceLoad.load(xx.class)

ServiceLoad<HelloInterface> loads = ServiceLoad.load(HelloInterface.class)

```

# dubbo 中的 SPI

dobbo 对 jdk 的标准SPI 进行了加强。

官方的文档如下：

>* JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
>* 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
>* 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。

# dubbo 的 SPI 具体实现

## @SPI 注解定义和使用

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {
    String value() default "";
}
```

dubbo 定义了一个 @SPI 的注解。

```java
@SPI("spring")
public interface Container {
    //...
}
```

对于 `Container` 来说，默认的实现就是 `spring`。我们在 `/dubbo-container/dubbo-container-spring/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.container.Container` 发现了配置文件：

```
spring=com.alibaba.dubbo.container.spring.SpringContainer
```

这个时候向dubbo框架说明。`Container` 的默认实现是 `spring`,而`spring`的具体实现是 `com.alibaba.dubbo.container.spring.SpringContainer` 这个类。了解了dubbo是怎么使用注解 @SPI 之后，我们就看看到底是怎么实现的。

## ExtensionLoader

ExtensinLoader 是 dubbo 实现 SPI 机制的核心类。在Ext中有两个 map：

```java
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

```  
EXTENSION_LOADERS 缓存了 class 和他对应的 ExtensionLoader。
EXTENSION_INSTANCES 缓存的是 class 和对应的类的实例。

在使用前会先从 map 中获取缓存，只有在获取不到class对应的缓存后，会创建对应的 ExtensionLoader 或者对应类的实例。接下来的源码中会有具体的分析。


```java

    private static final ExtensionLoader<Container> loader = ExtensionLoader.getExtensionLoader(Container.class);

```

