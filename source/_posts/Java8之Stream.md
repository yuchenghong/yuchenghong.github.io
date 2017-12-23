---
title: Java8之Stream
date: 2017-12-23 16:27:50
update: 2017-12-23 16:27:50
categories: java
tags: [java]
---
Java8中的Stream是对集合对象功能的增强。  
Stream API和Lambda表达式极大的提高对集合类操作编程效率和程序可读性。  

# 内部迭代
先看看外部迭代:  通过外部的迭代器来遍历集合。  
比如我们传统的遍历集合的方式就是外部迭代。  
传统的for循环，也是外部的迭代。 
```
Iterator<Artist> iterator = allArtists.iterator();
while(iterator.hasNext()) {
    Artist artist = iterator.next();
    //do something
}

for(Artist artist:allArtists){
    //do something
}

```

内部迭代: 不依赖于外面的迭代器，完全通过集合本身和函数式接口来把元素取出来. 

```
List<Integer> list = Arrays.asList(1,2,3,4,5,6,7,8);
list.forEach((Integer i) -> {
    System.out.println(i);
});
```

# 什么是Stream: 
Stream 是用函数式编程方式在集合类上进行复杂操作的工具。

# 简单的例子
上代码，直观点。  
统计list里面，"Lon"的数量： 
```
List<String> list = new ArrayList<>();
list.add("Lon");
list.add("SH");
list.add("BJ");
list.add("Lon");
long count = list.stream().filter(arts -> arts.equals("Lon")).count();
```

# Stream构成
1，源: 集合数据源  
2，零个或多个中间操作：比如上面的fileter()就是中间操作。  
3，终止操作：示例的count()  

```
stream.xxx().yyy().zzz().count()
```

Stream操作的分类  
1，惰性求值  
2，及早求值

在 Java 中调用一个方法， 计算机会随即执行操作： 比如， System.out.println("Hello World"); 会在终端上输出一条信息。 Stream 里的一些方法却略有不同。  
比如：

```
list.stream().filter(arts -> arts.equals("Lon"));
```
这行代码并未做什么实际性的工作， filter只刻画出了 Stream， 但没有产生新的集合。  
像filter 这样只描述 Stream，最终不产生新集合的方法叫作**惰性求值**；  
而像 count 这样最终会从 Stream 产生值的方法叫作**及早求值**。  

看下面的代码，由于使用了惰性求值，并不会打印任何信息。
```
list.stream().filter({
    System.out.println(arts);
    arts -> arts.equals("Lon");
});
```
当调用到终止操作.count()时，上面的程序才会有信息打印出来。  







# 流的操作
常见的操作可以归类如下。  
1，Intermediate中间操作： 
map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered  
2，Terminal终止操作： 
forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator  
3，Short-circuiting： 需要在遍历中途停止操作，比如查找第一个满足条件的元素。我们只需要处理流的一部分，而不是所有的流，以返回结果。这与计算使用and运算符链接的大型布尔表达式类似：只要一个表达式返回false，我们可以推断整个表达式是假的，而不计算它。  
下面是Short-circuiting操作列表：  
anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit  

# Stream方法列表
1, collect(toList())
由Stream值生成一个List, 是及早求值操作。

2, map:把 input Stream 的每一个元素，映射成 output Stream 的另外一个元素。

```
List<String> output = wordList.stream().
map(String::toUpperCase).//每个都转换成大写
collect(Collectors.toList());

List<Integer> nums = Arrays.asList(1, 2, 3, 4);
List<Integer> squareNums = nums.stream().
map(n -> n * n).//把每个元素平方
collect(Collectors.toList());
```


3, filter
filter 对原始 Stream 进行某项测试，通过测试的元素被留下来生成一个新 Stream。

```
Integer[] sixNums = {1, 2, 3, 4, 5, 6};
Integer[] evens = Stream.of(sixNums).filter(n -> n%2 == 0).toArray(Integer[]::new);

List<String> output = reader.lines().
 flatMap(line -> Stream.of(line.split(REGEXP))).
 filter(word -> word.length() > 0).
 collect(Collectors.toList());
```


4, flatmap
map 生成的是个 1:1 映射，每个输入元素，都按照规则转换成为另外一个元素。还有一些场景，是一对多映射关系的，这时需要 flatMap。

```
Stream<List<Integer>> inputStream = Stream.of(
 Arrays.asList(1),
 Arrays.asList(2, 3),
 Arrays.asList(4, 5, 6)
 );
Stream<Integer> outputStream = inputStream.flatMap((childList) -> childList.stream());
```
flatMap 把 input Stream 中的层级结构扁平化，就是将最底层元素抽出来放到一起，最终 output 的新 Stream 里面已经没有 List 了，都是直接的数字。

5, min/max/distinct: 最小值/最大值/去重复的值.

```
List<String> words = br.lines().
 flatMap(line -> Stream.of(line.split(" "))).
 filter(word -> word.length() > 0).
 map(String::toLowerCase).
 distinct().//去重复值
 sorted().
 collect(Collectors.toList());
```

6, reduce
这个方法的主要作用是把 Stream 元素组合起来。它提供一个起始值（种子），然后依照运算规则（BinaryOperator），和前面 Stream 的第一个、第二个、第 n 个元素组合。从这个意义上说，字符串拼接、数值的 sum、min、max、average 都是特殊的 reduce。例如 Stream 的 sum 就相当于

```
Stream<Integer> integers = Stream.of(4, 5, 6);
Integer sum = integers.reduce(5, (a, b) -> a+b); //起始值为5，求和操作, 结果是：20
//或
Integer sum = integers.reduce(0, Integer::sum);

sumValue = Stream.of(1, 2, 3, 4).reduce(Integer::sum).get(); // 求和，sumValue = 10, 无起始值
```

7, forEach
forEach 是 terminal 操作，因此它执行后，Stream 的元素就被“消费”掉了，你无法对一个 Stream 进行两次 terminal 运算。

```
stream.forEach(element -> doOneThing(element));
```


8, peek
对每个元素执行操作并返回一个新的 Stream

```
Stream.of("one", "two", "three", "four")
 .filter(e -> e.length() > 3)
 .peek(e -> System.out.println("Filtered value: " + e))
 .map(String::toUpperCase)
 .peek(e -> System.out.println("Mapped value: " + e))
 .collect(Collectors.toList());
```


9, findFirst
这是一个 termimal 兼 short-circuiting 操作，它总是返回 Stream 的第一个元素，或者空。

10, limit/skip
limit 返回 Stream 的前面 n 个元素；skip 则是扔掉前 n 个元素

```
map(Person::getName).limit(10).skip(3).collect(Collectors.toList());
```


11, sorted
对 Stream 的排序通过 sorted 进行，它比数组的排序更强之处在于你可以首先对 Stream 进行各类 map、filter、limit、skip 甚至 distinct 来减少元素数量后，再排序，这能帮助程序明显缩短执行时间。

```
List<Person> persons = new ArrayList();
 for (int i = 1; i <= 5; i++) {
 Person person = new Person(i, "name" + i);
 persons.add(person);
 }
List<Person> personList2 = persons.stream().limit(2).sorted((p1, p2) -> p1.getName().compareTo(p2.getName())).collect(Collectors.toList());
System.out.println(personList2);
```


12, Match
Stream 有三个 match 方法，从语义上说： 
allMatch：Stream 中全部元素符合传入的 predicate，返回 true  
anyMatch：Stream 中只要有一个元素符合传入的 predicate，返回 true  
noneMatch：Stream 中没有一个元素符合传入的 predicate，返回 true  
例如 allMatch 只要一个元素不满足条件，就 skip 剩下的所有元素，返回 false。

```
List<Person> persons = new ArrayList();
persons.add(new Person(1, "name" + 1, 10));
persons.add(new Person(2, "name" + 2, 21));
persons.add(new Person(3, "name" + 3, 34));
persons.add(new Person(4, "name" + 4, 6));
persons.add(new Person(5, "name" + 5, 55));
boolean isAllAdult = persons.stream().
 allMatch(p -> p.getAge() > 18);
System.out.println("All are adult? " + isAllAdult);
boolean isThereAnyChild = persons.stream().
 anyMatch(p -> p.getAge() < 12);
System.out.println("Any child? " + isThereAnyChild);
```

