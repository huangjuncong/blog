---
title: JAVA 8入门（二）流
date: 2017-10-24 19:15:04
tags: [Java,Java 8,stream]
---

## 1.简单使用
书接上回，我们这一讲要讨论 JAVA 8 的新的 API 流。如果我们有这样一个需求，需要挑选出菜谱里面卡路里小于1000，且卡路里排名前三的菜品的名称。

<!--more-->

如果使用`JAVA 7`的传统写法我们应该这样写：

```java
    public List<String> findDish(List<Dish> menu){
        List<Dish> lowCaloricDishes = new ArrayList<>();
        for (Dish dish : lowCaloricDishes) {
            if(dish.getCalories() < 1000 ){
                lowCaloricDishes.add(dish);
            }
        }

        Collections.sort(lowCaloricDishes, new Comparator<Dish>() {
            @Override
            public int compare(Dish o1, Dish o2) {
                return Integer.compare(o1.getCalories(),o2.getCalories());
            }
        });

        List<Dish> result = lowCaloricDishes.subList(0,3);
        List<String> resultName = new ArrayList<>();

        for (Dish dish : result) {
            resultName.add(dish.getName());
        }
        return resultName;
    }
```

通过上面的例子我们可以看到我们使用了很多的中间变量，来存储中介结果，lowCaloricDishes、result、resultName。相当繁琐。如果我们使用 `JAVA 8`的流的方式来实现代码是这样的:

```java

    public List<String> findDishWithSteam(List<Dish> menu){
        

        List<String> resultName = menu.stream()
                .filter(dish -> dish.getCalories()<1000)
                .sorted(Comparator.comparing(Dish::getCalories))
                .limit(3)
                .map(Dish::getName)
                .collect(Collectors.toList());
        
        return resultName;
    }

```

如果我们把 `steam()` 变换为 `parallelStream()`，整个操作就变成并行的。代码十分优雅，想到每天我们处理那么多的集合，反正我已经迫不及待的想使用上`JAVA 8了。


## 2. 流的定义
到底什么是流？书上给的定义是 *“从支持数据处理操作的源生成的元素序列”*。

* 元素序列 和集合一样，我们可以理解为是一堆有序的值。但是集合侧重的是数据，流侧重的是计算。
* 源 流会使用一个提供数据的源。比如 `menu.stream()`中，`meun`就是这个流的源。
* 数据处理操作 流的数据处理功能类似，数据库的操作，同时也支持函数式编程中的操作。

我们看看接口`java.util.stream.Stream`都有一些什么方法：

![stea](http://upload-images.jianshu.io/upload_images/4268675-5584795be6e3b7e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，之前我们在上一个例子里面使用的方法，`filter()`，`sorted()`，`limit()`，`map()` 的返回值也是一个流`Stream`，也就是说我们可以，把所有的操作串起来。这个是流的一个特点*流水线*

还有一个特点就是*内部迭代*，与集合的迭代不同，流的迭代不是显式的迭代。

## 3. 流的基本操作

### 1. 筛选和切片
1. 筛选filter()
    
    所谓筛选就是找出符合条件的元素，`filter()`接受一个返回`boolean`类型的函数。
    比如：
    
    ```java
        filter(dish -> dish.getCalories()<1000)
    ```
2. 去重distinct()

    去重的方法我们可以类比`SQL` 语句中的 distinct
3. 截断limit()

    同样类似 `SQL`里面的 limit，接受一个 Long 值。返回流中的前n个元素。 
4. 跳过skip()
    
    跳过前n个元素。很好理解
    
### 2. 映射

1. map()

    map对流中的每一个元素应用函数，可以理解为将元素转化为另一个元素。

    ```java
        .map(Dish::getName)
    ```

2. flatmap()

    flatmap方法就是把流中的每一个元素都装换为另外一个流，然后合并为一个流。
    
    ```java
List<List<String>> lists = new ArrayList<>();
        lists.add(Arrays.asList("apple", "click"));
        lists.add(Arrays.asList("boss", "dig", "qq", "vivo"));
        lists.add(Arrays.asList("c#", "biezhi"));  
    ```
    
    找出所有大于两个字符的元素：
    
    ```java
    
    lists.stream()
        .flatMap(Collection::stream)
        .filter(str -> str.length() > 2)
        .count();
    
    ```
    
    

### 3. 查找匹配
match、anyMatch、allMatch、noneMatch
以上方法都能返回一个boolean类型。

```java
boolean hasLowCalories = mune.stream().anyMatch(dish -> dish.getCalories()<1000)    
```

### 4. 归约
reduce(T，BinaryOperator<T>)
reduce 操作是 反复结合每一个元素，直到流被归约成一个值。其中：
* `T` 指的是初始值； 
* `BinaryOperator<T>` 两个元素结合起来获得一个元素，举个例子：
Lamdba: `(a,b)-> a + b`。

所以给定一个 `List<Integer>` 计算出和所有`int` 的和：

```java
int sum = list.stream().reduce(0,(a,b)-> a + b);
```
或者:

```java
int sum = list.stream().reduce(0,Integer::sum);
```


### 5. 收集数据
一顿操作之后，我们需要把数据收集起来。就需要使用`collect()`方法。

```java
Map<String, Integer> map = meun.stream()
        .collect(Collectors.toMap(Dish::getName, Dish::getCalories));
        
```
