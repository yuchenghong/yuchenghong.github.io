---
title: Java并发之线程管理
date: 2017-12-17 10:26:15
update: 2017-12-17 10:26:15
categories: java
tags: [java]
---

最近有朋友问到Java并发  
我觉得有必要对并发的知识做个总结  
从这篇开始，我会整理一系列的并发相关的 

关于线程的最基本的介绍(例如hello world), 这里就不说了，百度上有大把的
# 线程阻塞
sleep(): 指定一定的时间作为参数，线程在指定的时间内阻塞，时间结束后自动恢复到可执行状态, sleep状态不会释放锁。  
  
suspend() 和 resume()：  两个方法配套使用，suspend()使得线程进入阻塞状态，并且不会自动恢复，对应的resume() 被调用，才能使得线程重新进入可执行状态。  
用在等待另一个线程产生的结果的情形，A结果还没有产生，让B线程阻塞，A线程产生了结果后，调用 resume() 使B恢复。  
suspend阻塞时不会释放占用的锁。  
  
wait() 和 notify()：wait() 使得线程进入阻塞状态，对应的 notify() 被调用或者超出指定时间时线程重新进入可执行状态。  
wait状态会释放锁。
  
join()：
当前线程调用某个线程的这个方法时，它会暂停当前线程，直到被调用线程执行完成。  
如： A.jion(B), A线程会被阻塞，直到B线程完成.   
此外方法join (long milliseconds)，时间参数，等待相应的毫秒数

# 线程中断
Java提供中断机制来通知线程表明我们想要结束它。  
中断机制的特性是线程需要检查是否被中断，而且还可以决定是否响应结束的请求。  
**所以，线程可以忽略中断请求并且继续运行。**  
  
interrupt() 通知线程，我们想要结束他  
isInterrupted() 线程内通过这个方法检测到请求。 此时线程可以选择return或者抛异常中止; 也可以选择忽略这条消息。程序会继续执行。

如果线程在sleep()时，调用interrupt()  睡眠会立即中断 并马上抛出InterruptedException。   
  
# 处理线程异常
run()里不能抛出异常; 否则线程会被中断  
对于检查型异常，使用try catch捕获处理  
对于运行时异常，实现UncaughtExceptionHandler来处理运行时异常。

```
public class ExceptionHandler implements UncaughtExceptionHandler{
     public void uncaughtException(Thread t, Throwable e) {
          System.out.printf("An exception has been captured\n");
     }
}
```

然后调用:

```
Thread thread=new Thread(task);
thread.setUncaughtExceptionHandler(new ExceptionHandler());
thread.start();
```


# 本地线程变量
如果你创建一个类对象，实现Runnable接口，然后多个Thread对象使用同样的Runnable对象，全部的线程都共享同样的属性。这意味着，如果你在一个线程里改变一个属性，全部的线程都会受到这个改变的影响。
有时，你希望程序里的各个线程的属性不会被共享。 

```
private ThreadLocal<Date> safeDate = new ThreadLocal<Date>(){
    protected Date initialValue(){
        return new Date();
    }
};
```

本地线程变量为每个使用这些变量的线程储存属性值。可以用 get() 方法读取值和使用 set() 方法改变值。 如果第一次你访问本地线程变量的值，如果没有值给当前的线程对象，那么本地线程变量会调用 initialValue() 方法来设置值给线程并返回初始值。

# 线程组



```
ThreadGroup threadGroup = new ThreadGroup("Group1");
Thread thread = new Thread(threadGroup, new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});
thread.start();
//通过这种方法可以看group里面的所有信息  
System.out.printf("Number of Threads: %d\n", threadGroup.activeCount());  
System.out.printf("Information about the Thread Group\n");
threadGroup.list(); //会打印出所有对象信息
```

activeCount() 和 enumerate() 方法来获取线程个数和与ThreadGroup对象关联的线程的列表。我们可以用这个方法来获取信息， 例如，每个线程状态。
```
Thread[] threads=new Thread[threadGroup.activeCount()];
threadGroup.enumerate(threads);
for (int i=0; i<threadGroup.activeCount(); i++) {
    System.out.printf("Thread %s: %s\n",threads[i].getName(),threads[i].getState());
}
```

# 参考资料
并发中文网: http://ifeve.com/

