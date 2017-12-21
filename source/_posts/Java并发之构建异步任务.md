---
title: Java并发之构建异步任务
date: 2017-12-21 22:17:15
update: 2017-12-21 22:17:15
categories: java
tags: [java]
---

有个小朋友表示我写的并发文章，不够细，看了还是不太明白。  
我的写作水平实在有限，泪奔。  
后面我尽量写的详细简单，尽量多讲点例子。  

异步对应的就是同步。  
同步的程序大家写的非常的多，同步就是指程序对所有的操作都是串行的处理。一系列的任务A,B,C,D...必须一个一个的处理。  
而异步则是可以并发同时处理多个任务。  

举个栗子吧。  
某个网站的用户提交注册需要以下几步：  
1，调用用户服务，写入数据库-需要1s
2，发送邮件-需要1s
3, 发送短信-需要1s

如果是同步的程序，串行执行，则用户一共需要等代3s  
但是我们注意到，这三个任务相互之间没有依赖关系，可以改成异步并发的来执行，则用户等待的时间一共只需要1s.  

异步通常代表着更高的性能和更好的系统资源利用率。  

有几种方法可以构建异步的任务  
1， Thread和Runnable接口。  
Runnable是最基本任务表现形式，但不能返回值。 

```
Thread thread1 = new Thread(
    new Runnable(){
        public void run(){
            //写入数据库
        }
    }
);
thread1.start();
Thread thread2 = new Thread(
    new Runnable(){
        public void run(){
            //发送邮件
        }
    }
);
thread4.start();
Thread thread3 = new Thread(
    new Runnable(){
        public void run(){
            //发送邮件
        }
    }
);
thread3.start();
```


2， Future  
Future表示一个任务的生命周期：  
1）提供了相应的方法来判断任务是否已经完成或者取消；  
2）还能获取任务的结果；  
3）可以取消任务。  
 
最常用的是通过ExecutorService的submit方法提交任务，会返回一个Future。  
ExecutorService的submit方法可以提交Runnable和Callable任务；  
对于Callable任务，Future的get方法可以获得返回值。  

```
ExecutorService executorService = Executors.newFixedThreadPool(10);
//定义Callable, 返回一个结果
Callable<Integer> task = ()->{
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 5;
};
Future<Integer> future = executorService.submit(task);
Integer result = future.get();
System.out.println(result);
```

在实际的开发中，对于耗时比较长得操作，都可以考虑异步的方式来执行。  
一方面可以大大提高资源利用率和性能；  
并且对于用户交互的任务，可以大大减少用户的等待时间，提升用户体验。  

