---
title: Java并发之同步工具
date: 2017-12-19 22:46:26
update: 2017-12-19 22:46:26
categories: java
tags: [java]
---

# BlockingQueue阻塞队列  
阻塞队列一般用于生产-消费模式：生产者把数据放入队列，消费者处理数据  
put()方法插入数据，如果队列已满，阻塞，直到有可用空间； take()方法获取数据，如果队列为空，阻塞，直到有可用元素；    
offer(),poll()也是插入获取元素，和put,take不同的是，offer,poll不会阻塞，操作失败时，返回失败状态； 
offer,poll方法，支持定时的参数。  
  
BlockingQueue是接口，具体的实现有：  
ArrayBlockingQueue：有界的阻塞队列。  
DelayQueue：对元素进行持有直到一个特定的延迟到期。  
LinkedBlockingQueue：链式结构，如果没有定义上限，将使用 Integer.MAX_VALUE 作为上限。  
PriorityBlockingQueue：无界的并发队列，插入到 PriorityBlockingQueue 的元素必须实现 java.lang.Comparable 接口。元素的排序就取决于Comparable实现  
SynchronousQueue：只能够容纳单个元素。  
  
简单看一个例子  

```
BlockingQueue queue = new ArrayBlockingQueue(1024);
queue.put("1");
queue.put("2");
queue.put("3");

System.out.println(queue.take());
System.out.println(queue.take());
System.out.println(queue.take());

```



# CountDownLatch闭锁
多个线程等待，直到达到最终状态。  
好比一扇门， 达到结束状态之前，这扇门一直关闭，没有线程能通过，达到结束状态时，门打开，所有线程通过。  
构造函数传入一个int n计数器；  
await()方法会阻塞当前线程；  
调用cuntDown()方法，n会减1，直到n变为0时，await()方法不再阻塞当前线程；  

```
CountDownLatch latch = new CountDownLatch(3);//定义N=3

for(int i=0;i<3;i++){
    new Thread(
        new Runnable(){
            public void run(){
                latch.await();//阻塞，直到调用3次countDown()
            }
        }
    ).start();
}

Thread.sleep(1000);
latch.cuntDown();
Thread.sleep(1000);
latch.cuntDown();
Thread.sleep(1000);
latch.cuntDown();//执行3次后, 上面的线程不再阻塞

```



# CyclicBarrier栅栏  
所有线程必须等待的一个栅栏，直到所有线程都到达这里，然后所有线程才可以继续做其他事情。  
构造函数传入int n计数器;   
await()方法会阻塞当前线程, 每调一次await()方法,  n-1，直到n=0时，所有await()不再阻塞.   
此外还有个带Runnable接口参数的构造函数，当n=0会执行这个线程的方法。  
  

```
CyclicBarrier barrier = new CyclicBarrier(3);//定义n=3的栅栏
for(int i=0;i<3;i++){
    new Thread(
        new Runnable(){
            public void run(){
                barrier.await();//阻塞，直到足够的线程到达这里: 调用3(n)次await()
            }
        }
    ).start();
}
```



# Semaphore信号量
Semaphore是一个控制访问多个共享资源的计数器。可以控制并发线程数。  
当一个线程想要访问某个共享资源：  
1，首先，它必须获得semaphore。尝试获得semaphore，如果semaphore的内部计数器的值大于0，那么semaphore减少计数器的值并允许访问共享的资源。  
2，如果semaphore的计数器的值等于0，那么semaphore让线程进入休眠状态一直到计数器大于0。计数器的值等于0表示全部的共享资源都正被线程们使用，所以此线程想要访问就必须等待。  
3，当线程使用完共享资源时，他必须放出semaphore为了让其他线程可以访问共享资源。这个操作会增加semaphore的内部计数器的值。

3个步骤使用semaphore来实现临界区，并保护共享资源的访问：  
1. 首先， 你要调用acquire()方法获得semaphore。  
2. 然后， 对共享资源做出必要的操作。  
3. 最后， 调用release()方法来释放semaphore。  

Semaphore类有另2个版本的 acquire() 方法：
1. acquireUninterruptibly()：acquire()方法是当semaphore的内部计数器的值为0时，阻塞线程直到semaphore被释放。在阻塞期间，线程可能会被中断，然后此方法抛出InterruptedException异常。而此版本的acquire方法会忽略线程的中断而且不会抛出任何异常。
2. tryAcquire()：此方法会尝试获取semaphore。如果成功，返回true。如果不成功，返回false值，并不会被阻塞和等待semaphore的释放。接下来是你的任务用返回的值执行正确的行动。

Semaphore公平性
Semaphore类的构造函数容许第二个参数。这个参数必需是 Boolean 值。如果你给的是 false 值，那么创建的semaphore就会在非公平模式下运行。如果你不使用这个参数，是跟给false值一样的结果。如果你给的是true值，那么你创建的semaphore就会在公平模式下运行。


```
Semaphore semaphore=new Semaphore(1);
semaphore.acquire();
semaphore.release();
```

# Exchanger交换机
两个线程可以进行互相交换对象  

```
Exchanger exchanger = new Exchanger();
new Thread(
    new Runnable(){
        public void run(){
            String obj = "A";
            String previous = obj;
            obj = (String)exchanger.exchange(obj);
            System.out.println(Thread.currentThread().getName() +" exchanged " + previous + " for " + obj);
        }
    }
).start();

new Thread(
    new Runnable(){
        public void run(){
            String obj = "B";
            String previous = obj;
            obj = (String)exchanger.exchange(obj);
            System.out.println(Thread.currentThread().getName() +" exchanged " + previous + " for " + obj);
        }
    }
).start();
```
输出结果为：
Thread-1 exchanged B for A
Thread-0 exchanged A for B


# Phaser
Phaser类运行阶段性的并发任务。当某些并发任务是分成多个步骤来执行时，那么此机制是非常有用的。Phaser类提供的机制是在每个步骤的结尾同步线程，所以除非全部线程完成第一个步骤，否则线程不能开始进行第二步。

```
Phaser phaser = new Phaser(3); //创建 含3个参与者的 Phaser 对象。
phaser.arriveAndAwaitAdvance();  //会被阻塞，直到达到条件(这里3个线程全部调用到这个方法时, 焕醒)
phaser.arriveAndDeregister(); //撤销对phaser线程的注册
phaser.isTerminated();   //当phaser 有0个参与者时，它进入termination状态
```




