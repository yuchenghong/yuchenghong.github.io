---
title: Java并发之执行器和线程池
date: 2017-12-19 23:31:22
update: 2017-12-19 23:31:22
categories: java
tags: [java]
---
# 执行器Executor
用来管理Thread对象，简化并发编程  
Executor的子接口是ExecutorService  
ExecutorService的实现类实际上就是**线程池**  
  
简单的例子:
```
ExecutorService executorService = Executors.newFixedThreadPool(4);
executorService.execute(new Runnable() {
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
});
executorService.shutdown();
```
## ExecutorService实现类
ThreadPoolExecutor  
ScheduledThreadPoolExecutor  

## 创建ExecutorService
Executors工厂类来创建：
```
ExecutorService executorService1 = Executors.newSingleThreadExecutor();
ExecutorService executorService2 = Executors.newFixedThreadPool(10);
ExecutorService executorService3 = Executors.newScheduledThreadPool(10);
```
## 使用ExecutorService
execute(Runnable)：执行Runnable  
submit(Runnable)：执行Runnable，返回Future，这个Future可以用来检查Runnable是否已经执行完毕。  
submit(Callable)：执行Callable，返回Future，这里的Future可以获得Callable的返回数据。  

## Future和Callable
Callable：和Runnable接口很类似。此接口有一个call()方法。在这个方法中，实现任务逻辑并返回一个结果。  
Future：表示一个任务的生命周期，判断任务是否完成，获取Callable任务的返回结果。  

```
ExecutorService executorService = Executors.newFixedThreadPool(1);
Callable<Integer> task = ()->{
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 5;//返回5
};
Future<Integer> future = executorService.submit(task);
Integer result = future.get();
System.out.println(result);
```
## ExecutorService关闭
shutdown()：并不会立即关闭，但它将不再接受新的任务，而且一旦所有线程都完成了当前任务的时候， ExecutorService 将会关闭。  
shutdownNow()：立即停止所有执行中的任务，并忽略掉那些已提交但尚未开始处理的任务。无法保证正在执行的任务的正确性。  

# 线程池
ThreadPoolExecutor是ExecutorService接口的一个实现。  
一般我们可以用Executors工厂类来创建线程池，其用法和上面的讲的例子一样。   
同时我们也可以使用ThreadPoolExecutor的构造方法

```
ExecutorService threadPoolExecutor =
    new ThreadPoolExecutor(
        corePoolSize,
        maxPoolSize,
        keepAliveTime,
        TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>()
    );
```



