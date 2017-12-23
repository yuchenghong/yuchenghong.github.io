---
title: Java8之函数式接口
date: 2017-12-23 11:46:42
update: 2017-12-23 11:46:42
categories: java
tags: [java]
---
在我的上一篇java8基础入门的文章中说了函数式接口的基本概念，这里介绍下JDK提供的一些重要的函数式接口:
# 重要的函数式接口
接口名称 | 参数 | 返回值
---|---|---
Function<T, R> | T, R | R
Consumer<T> | T | void
BiFunction<T, U, R> |T,U | R 
Predicate<T> | T | boolean
Supplier | None | T

# Function 接口: 
官方文档是这样介绍的: 输入一个参数，得到一个结果。  
初一看不太好理解。下面慢慢说来:    
1，apply方法, 抽象的方法。  
java8之前，我们代码老的方式，是先定义好行为，再调用。  
现在的方式，行为不需要先定义，通过方法引用的方式，把行为作为参数传递  
如老的方式可以这样定义一个方法，然后调用： 

```
public int method1(int a){
    return 2*a;
}
```
采用java8新的方式:

```
Function<Integer,Integer> function = value->2*value;
System.out.println(function.apply(100)); //打印返回的结果
```

2，compose方法, 非抽象的default方法.  
根据源代码，我们可以知道这个方法的作用：  
当前function实例和另一个function参数进行组合, 先应用参数的apply, 再应用当前的apply

```
default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
    Objects.requireNonNull(before);
    return (V v) -> apply(before.apply(v));
}
```
看个例子就更清楚了：

```
Function<Integer,Integer> function1 = value->2*value;
Function<Integer,Integer> function2 = value->3+value;
//看下面的组合：先应用参数的apply,也就是function2的操作=5+3=8; 再应用function1的操作=2*8=16
Integer result = function1.compose(function2).apply(5);
System.out.println(result);
```
程序输出的最终结果为：16

3，andThen方法:  
也是组合操作，先应用当前的apply, 再应用参数的apply。  
```
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t));
}
```
看个例子, 注意和上面的compose的结果对比一下。

```
Function<Integer,Integer> function1 = value->2*value;
Function<Integer,Integer> function2 = value->3+value;
//先应用当前对象的的apply,也就是function1的操作=2*5=10; 再应用参数function2的操作=3+10=13
Integer result = function1.andThen(function2).apply(5);
System.out.println(result);
```
程序输出的最终结果为：13


# Consumer<T>接口:  
1，官方文档的说明：抽象方法void accept(T t)，接受一个参数，没有返回结果，有可能会产生副作用。  
可以这么理解呢：在参数T上执行操作,不返回值,有可能改变传入的T的值。  

```
Consumer<String> consumer = str -> System.out.println(str);
consumer.accept("hello world!");

```
2，Consumer除了accept的抽象方法外，还有个default方法: andThen  
这个方法有什么用呢？我们看一下源码:
```
default Consumer<T> andThen(Consumer<? super T> after) {
    Objects.requireNonNull(after);
    return (T t) -> { accept(t); after.accept(t); };
}
```
接受一个Consumer返回一个Consumer；  
先调用当前Consumer对象的accept方法，再调用参数的accept方法；  
相当于把两个Consumer的操作**组合**起来。  

```
Consumer<String> consumer1 = str -> System.out.println("hello "+str);
Consumer<String> consumer2 = str -> System.out.println("bye "+str);
consumer1.andThen(consumer2).accept("Tom");
```
输出的结果是：  
hello Tom  
bye Tom  

# BiFunction接口:
1，先看jdk说明：输入两个参数，得到一个结果。  

```
BiFunction<Integer,Integer,Integer> biFunction = (value1,value2) -> value1 + value2;//两个数相加，返回结果
Integer biresult = biFunction.apply(2,3);
System.out.println(biresult);
```
结果为: 5

2，同样有一个andThen方法:
接收Function参数，返回BiFunction，把BiFunction和Function操作组合：  
先调用BiFunction的操作，再调用Function的操作。下面是源码：
```
default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t, U u) -> after.apply(apply(t, u));
}
```
看个例子：

```
Function<Integer,Integer> function3 = value->3*value;
BiFunction<Integer,Integer,Integer> biFunction3 = (value1,value2) -> value1 + value2;
//调用BiFunction的操作=4+5=9；在调用Function操作：3*9=27
Integer biAndThenResult = biFunction3.andThen(function3).apply(4,5);
System.out.println(biAndThenResult);
```
结果为: 27  

这里就有个问题了，为什么andThen方法参数是Function而不是BiFunction？  
仔细分析就能很快得出结论：那是因为BiFunction返回值只有一个值, 而BiFunction必须接收两个参数，所有没有办法把两个BiFunction进行足额组合。 只能传给Function，和Function操作进行组合。    

3, 为什么没有compose方法? 
我们的Function都有compose方法，为什么BiFunction没有呢?   
我们知道compose方法是先应用参数的apply, 在应用当前对象的apply, 问题就来了：  
先执行参数的apply, 其返回值一定只有一个, 不满足BiFunction要两参数的条件。  

# Predicate
判断输入的对象是否符合某个条件。  
1，抽象test方法：
```
//判断输入的数值是否大于100
Predicate<Integer> predicate = value -> value>100;
System.out.println(predicate.test(90));
System.out.println(predicate.test(101));
```
结果为：  
false  
true  
2，and方法：两个条件组合 && 判断

```
default Predicate<T> and(Predicate<? super T> other) {
    Objects.requireNonNull(other);
    return (t) -> test(t) && other.test(t);
}
```

3，or方法：两个条件 || 判断

```
default Predicate<T> or(Predicate<? super T> other) {
    Objects.requireNonNull(other);
    return (t) -> test(t) || other.test(t);
}
```

4，negate方法：非条件的判断

```
default Predicate<T> negate() {
    return (t) -> !test(t);
}
```
and,or,negate方法一起看个例子：

```
Predicate<Integer> predicate1 = value -> value>100;
Predicate<Integer> predicate2 = value -> value<1000;
System.out.println(predicate1.and(predicate2).test(300)); // value 大于100 and value <1000
System.out.println(predicate1.or(predicate2).test(300));// value 大于100 or value <1000
System.out.println(predicate1.negate().test(300));//value 不 大于100

```
结果：  
true  
true  
false  

# Supplier
只有一个抽象方法: T get();  
不接受参数，返回一个结果。  
我们上面看了那么多实例，这个接口应该很容易理解了。  
下面例子，打印hello world：
```
Supplier supplier = () -> "hello world";
System.out.println(supplier.get());
```

本文仅仅是简单介绍了一下jdk自带的常用的函数式接口。  
更深入的东西还是需要大家多读多写代码。
