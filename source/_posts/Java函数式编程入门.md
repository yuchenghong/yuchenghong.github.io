---
title: Java函数式编程入门
date: 2017-12-21 20:30:47
update: 2017-12-21 20:30:47
categories: java
tags: [java]
---
虽然最近java9都已经发布了。但是目前很多同学连java8都没有用到。  
java8增加了很多有用的特性，对于大家的编程还是有很大帮助的。没有用过java8的同学还是非常有必要了解一下这些新特性的。  
  
java8有个很重要的改变：增加了函数式编程(Lambda表达式)。  
相比以前的Java新特性，lambda表达式对于从没接触函数式编程的Java程序员来说，并不是很好理解。因此我这儿特意一些分享，希望能对没有用过的同学有所帮助。  
  
为什么要有lambda表达式? 引用大师的原话"未来的编程语言将逐渐融合各自的特性，而不存在单纯的声明式语言（如之前的Java）或者函数编程语言。将来声明式编程语言借鉴函数编程思想，函数编程语言融合声明式编程特性...这几乎是一种必然趋势"。

# 什么是Lambda表达式
- **把函数当成参数在java方法中传递.**  
  
以往我们的java方法只能是传递一个值或者引用， 如: 

```
test.hello("hello world");//参数只能是传递值
```
而在lambda表达式里，可以这样:  

```
Thread thread = new Thread(() -> System.out.println("hello world"));
thread.start();
```
new Thread()参数传递的就是一个行为，打印"hello world". 

- **lambda表达式必须依附于一类特别的对象类型-函数式接口FunctionalInterface**。 比如上面的new Thread()方法就接收一个Runnable借口做为参数，在Java8中Runnable接口就是一个函数式接口。 
- **传递行为，而不是值;**
- **提升抽象层次，API重用性更好。**

# 第一个例子

```
//1. 定义一个函数式接口(后面会详细讲解函数式接口)
@FunctionalInterface
interface Interface1 {
    void test();
}

//2. 定义一个方法, 接受Interface1作为参数
public void testMethod(Interface1 interface1){
    interface1.test();
}

//3. 调用上面定义的方法: 注意，此时我们并没有实现Interface1，但可以正确执行代码
public static void main(String[] args){
    Test test = new Test();
    test.testMethod(() -> System.out.println("hello world"));
}
```
main方法里面，调用test.testMethod()方法时，传递一个行为：打印"hello world"。 
这就是一个最基本的lambda表达式的例子.  


# lambda表达式的基本结构:
(argument) -> {body}  
如
(arg1, arg2...) -> {}  
(type1 arg1,type2 arg2...) -> {body}  
示例说明:  
(int a,int b) -> {return a+b}  
() -> System.out.println("Hello world");  
(String s) -> {System.out.println(s);}  
() -> "Hello world"  
() -> {return "Hello world";}  
1. 一个lambda表达式可以有零个或多个参数  
2. 参数的类型可以明确声明，也可以根据上下文来推断。(int a) 与 (a)
3. 所有参数需要包含在括号内，参数之前用逗号隔开
4. 括号代表参数集为空。例如() -> "Hello world"
5. 当只有一个参数，类型可以推导，()可以省略。a-> {body}
6. lambda表达式的主体可以包含零或者多条语句
7. 如果lambda表达式主体只有一条语句, {}可以省略
8. 如果lambda表达式的主体包含一条以上语句, 则表达式必须包含在{}中。匿名函数返回类型与代码块的返回类型一致


# 函数式接口funcationalInterface: 
1, 一个接口有且仅有一个抽象方法称为函数式接口。  
2, 如果我们在某个接口上声明了functionalInterface， 那么编译器就会按照函数式接口的定义来要求接口，不满足定义的就会报错  
3, 如果某个接口只有一个抽象方法，但我们并没有给该接口声明functionalinterface注解，那么编译器依旧会将该接口看作是函数式接口  
4, 如果接口中的一个方法是覆写的java.lang.Object的方法，这个方法不计划在函数式方法的个数内.   
5, 函数式接口可以通过Lambda表达式,方法引用,构造器引用来实现.  
6，注意在java8，接口可以包含实现的default方法. 如List接口里就有这样一个default方法。  

```
default void replaceAll(UnaryOperator<E> operator) {
    Objects.requireNonNull(operator);
    final ListIterator<E> li = this.listIterator();
    while (li.hasNext()) {
        li.set(operator.apply(li.next()));
    }
}
```
lambda表达式最基本的入门就讲完了。后面逐步深入介绍。


