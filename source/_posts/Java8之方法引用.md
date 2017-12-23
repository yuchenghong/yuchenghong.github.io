---
title: Java8之方法引用
date: 2017-12-23 21:05:09
update: 2017-12-23 21:05:09
categories: java
tags: [java]
---

在Lambda表达式里，经常会有这样的代码

```
artist -> artist.getName()
```
就是lambda表达式里面调用了一个**已存在的方法**: getName()  
java8可以直接通过方法引用来简写lambda表达式中已经存在的方法，这种特性就叫方法引用。

```
Artist::getName
```
::前面是对象，后面是方法名称。  

# 方法引用语法
Classname::methodName  


# 分类
方法引用有四种类型:  
类名::静态方法名;  
对象引用名::实例方法名;  
类名::实例方法名;  
构造方法，类名::new  
  
下面分别来介绍

## 1，类名::静态方法名

```
//Integer对象有一个静态方法
public static int compare(int x, int y) {
    return (x < y) ? -1 : ((x == y) ? 0 : 1);
}

//普通lambda调用
List<Integer> integers = Arrays.asList(23,12,100,98);
integers.sort((arg1,arg2) -> Integer.compare(arg1,arg2));
System.out.println(integers);

//方法引用调用
List<Integer> integers = Arrays.asList(23,12,100,98);
integers.sort(Integer::compare);
System.out.println(integers);
```

## 2，对象引用名::实例方法名

```
//新创建一个类，可以对Integer进行排序
public class IntegerComparator {
    public int compare(Integer arg1,Integer arg2){
        return arg1-arg2;
    }
}

//普通lambda调用
IntegerComparator integerComparator = new IntegerComparator();
List<Integer> integers = Arrays.asList(23,12,100,98);
integers.sort((arg1,arg2)->integerComparator.compare(arg1,arg2));

//方法引用调用
IntegerComparator integerComparator = new IntegerComparator();
List<Integer> integers = Arrays.asList(23,12,100,98);
integers.sort(integerComparator::compare);

```

## 3，类名::实例方法名

```
//在看Integer对象有这样非静态的方法
public int compareTo(Integer anotherInteger) {
    return compare(this.value, anotherInteger.value);
}

//普通lambda调用
List<Integer> integers = Arrays.asList(23,12,100,98);
integers.sort((arg1,arg2) -> arg1.compareTo(arg2));

//方法引用调用
integers.sort(Integer::compareTo);

```

## 4，构造方法，类名::new

```
public class MethodReference {
    //定义一个方法
    public String getResult(String string,Function<String,String> function){
        return function.apply(string);
    }
    
    public static void main(String[] args) {
        MethodReference methodReference = new MethodReference();
        //普通lambda调用
        String result = methodReference.getResult("hello 1", (arg) -> new String(arg));
        System.out.println(result);
        
        //方法引用调用
        String result2 = methodReference.getResult("hello 2", String::new);
        System.out.println(result2);
    }
}

```
程序结果：  
hello 1  
hello 2


