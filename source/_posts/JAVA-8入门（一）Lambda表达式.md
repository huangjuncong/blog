---
title: JAVA 8入门（一）Lambda表达式
date: 2017-10-24 19:09:08
tags: [Java,Java 8,Lambda]
---

机房迁移以后终于可以用上 `Java 8`了，本教程将会分为三个方面介绍`Java 8` 的新特性。首先给大家介绍 `Java 8` 的Lambda 表达式。

<!--more-->

## 1. 让代码更灵活
作为程序员，每天除了写代码，最重要的事情就是吃饭了，为了吃饭，我们设计了一个Dish 对象，代码如下：

```java
public class Dish {

    private final String name;
    private final boolean vegetarian;
    private final int calories;
    private final Type type;
    
    public enum Type {MEAT,FISH,OTHER}
    
    //省略get set方法
}
```

作为一个减肥人士，寻求医生建议。医生说，低卡路里饮食，比较健康，为了找出卡路里低于1000的菜品。于是就有了一下代码：

```java
public static List<Dish> filterDish(List<Dish> dishes){
    List<Dish> healthDishes = new ArraryList<>();
    for(Dish dish: dishes){
        if(dish.getCalories()<1000){
            healthDishes.add(dish)
        }
    }
}
```
后来医生说，不只卡路里要低，而且肉就不要吃了，吃素比较有利于健康，于是含泪写了以下代码：

```java
public static List<Dish> filterDish(List<Dish> dishes){
    List<Dish> healthDishes = new ArraryList<>();
    for(Dish dish: dishes){
        if(dish.getCalories()<1000 && dish.getVegerarian()){
            healthDishes.add(dish)
        }
    }
}
```

不能吃肉哪憋得住，于是医生又说你可以吃一点鱼。最为一个有骨气的程序员，已经不想去迎合~~（产品经理了）~~医生去修改代码了？有没有什么办法，能快速找出健康食物，万一哪天减肥成功了，又能吃肉了也不用去修改代码？
于是我们写这样一段代码：

```java
public static List<Dish> filterDish(List<Dish> dishes,int calorites,boolean isMeat,Type type){
    List<Dish> healthDishes = new ArraryList<>();
    for(Dish dish: dishes){
        if(dish.getCalories()<calorites 
            && dish.getVegerarian() == isMeat
            && dish.getType() == type ){
            healthDishes.add(dish)
        }
    }
}
```

需求是满足了，但是作为一个有品位的程序员肯定不允许这样代码出现，实在过于繁琐了。万一再加入一个条件怎么办？
我们可以考虑将医生的医嘱作为一个方法传入我们的filerDish这个方法，医生说啥就是啥，不必要自己封装一个方法来响应医生的要求？于是我们这么考虑:

首先规定一个接口叫”医生说”：

```java
public interface DoctorSaid{
    boolean test(Dish dish);
}
```

我们挑选菜品的时候这样写：

```java
public static List<Dish> filterDish(Dish dishes,DoctorSaid doctorSaid){
    List<Dish> lowCaloriesDishes = new ArraryList<>();
    for(Dish dish: dishe){
        if(doctorSaid.test(dish) ){
            healthDishes.add(dish)
        }
    }
}
```

如果医生说吃 1000 卡路里一下的食物，我们实现一个1000 以下卡路里的食物：

```java
public boolean DoctorSaidLowCalorites implements DoctorSaid{
    boolean test(Dish dish){
        return dish.getCalorites() < 1000;
    }
}
```

这样我们只用这样调用filterDish就解决问题了：

```java
List<Dish> dishes = filterDish(dishes,new DoctorSaidLowCalorites());
```

问题来了，对于善变的~~（产品经理）~~ 医生，总是不能提前准备好所有的接口实现？
这个时候我们就可以使用`JAVA`的匿名了内部类来使用这个挑选菜品的方法更加灵活。于是我们就有这样的代码了

```java 
List<Dish> dishes = filterDish(dishes,new DoctorSaid(){
    return dish.getCalorites() < 1000;
});
```

稍微好了一点，但是匿名内部类还是有一个不好的地方，就是太啰嗦了其实核心代码就是`dish.getCalorites() < 1000` 为什么我们要写那么多代码？这个也是java老被诟病的地方,代码十分繁琐。
终于我们的Lambda表达式要出场了：

```java
List<Dish> dishes = filterDish(dishes, (Dish dish)-> dish.getCalorites() < 1000);
```

现在看上去好多了，但是作为一个程序员还是很贪心啊，现在只能过滤 `Dish` 能不能再抽象一点呢？当然啊，看这个：

```java
public interface Predicate<T>{
    boolean test(T t);
}

public static <T> List<T> filter(List<T> list, Predicate<T> p){
    List<T> result = new ArrayList<>();
    for(T e: list){
        if(p.test(e)){
            result.add(e);
        }
    }
    return result;
}
```

使用泛型让我们的代码更加通用。随便产品经理改需求，从 List 里面按规则过滤符合要求的需求不用多写代码都能搞定了。这时候你是不是觉得胸前的红领巾更加鲜艳了？

## 2. 实际应用：

作为一个招聘网站的程序员，一定有很多将职位列表排序的需求，比如按照更新时间将职位列表排序我们可以这么写：

```java

jobList.sort((JobInfo job1,JobInfo job2)->job1.getUpdateTime.compareTo(job2.getUpdateTime);

```

或者作为高端程序员的多线程可以这样写：

```java
    Thread t = new Thread(()-> System.out.println("Hello world"));
```

## 3. 近距离观察Lambda
### 1. 什么是Lambda
所以通过上面例子我们尝试定义一下Lambda 表达式是什么？

Lambda表达式为简洁的表示可传递的匿名函数的表达式的一种方式。分开来说： 

* 匿名：没有必要给他取一个函数名称。
* 简洁：相对于匿名内部内不需要写很多模板代码
* 传递：可以做为参数传递给方法
* 函数：不属特定类。但是和方法一样，有参数，函数主题，返回值，有时候还能抛异常。

标准的Lambda表达式:

```
(Dish dish) -> dish.getCalories() > 1000;
```

从上面的标准的表达式：一共三个部分组成
    * 参数：
    * 箭头：
    * Lamdba主体：也就是函数的主体

### 2. 什么时候使用Lambda
* 函数式接口：

所谓函数式接口，我们可以理解为就是只有一个方法的接口。

* 函数描述符：

函数式接口的签名基本上就是Lambda表达式的签名。我们降这种抽象方法叫做函数描述符。举个例子，Ruannable 方法就可以看做一个什么都不接受，什么都不返回的函数。这个时候我们可以发现传入的 Lambda 函数为 `()->void`.

### 3. Lambda的类型推断

Java编译器会根据上下文推断使用什么函数式接口来配合Lambda表达式，比如我们之前的例子，以下两种写法都是正确的：

```java
(Dish dish) -> dish.getCalories() > 1000;

dish -> dish.getCalories() > 1000;
``` 

第二个语句并没有显式的制定```dish```的类型是```Dish```，编译器也能正确的编译代码。

### 4.方法的引用

在Lambda中我们可以利用方法的引用来重复使用。可以认为是一个Lamdba带来的语法糖🍬。

对于之前我们为职位排序的例子：

```java
jobList.sort((JobInfo job1,JobInfo job2)->job1.getUpdateTime.compareTo(job2.getUpdateTime);
```

我们可以改写为

```java
jobList.sort(comparing(JobInfo::getUpdateTime));
```

反正我第一次看到这种写法也不禁感叹，这也行？还有这种操作？

我们就来看看 JDK 8 中对于方法的引用有以下四种类型：

* static 静态方法的引用，这个没啥好说的，语法就是 ```(ClassName:staticMethod)```。

* 任意类型的实例方法的引用，比如 String 方法的 length 方法：

```java
(String s1) -> s1.length()

(String::length)
```

* 现有对象的实例方法的引用：

```java
()-> s.length()

(s::length)
```

* 构造函数的引用：直接上代码也很好理解```(Dish::new)```。

#### 5.现成的函数式接口

JDK 8 中已经包含了若干现成的函数式接口。在```java.util.function```中。包括```Predicate<T>```，```Function<T,R>```，```Consumer<T>```。大家可以直接查看源码，在这里就不讲解了。


第一部分教程结束了，请大家期待《JAVA 8入门（二） 数据流的操作》