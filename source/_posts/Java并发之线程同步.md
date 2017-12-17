---
title: Java并发之线程同步
date: 2017-12-17 11:19:13
update: 2017-12-17 11:19:15
categories: java
tags: [java]
---

# Synchronized关键字
1，用于方法上  
2，用于代码块上: 必须通过一个对象引用作为参数。只有一个线程可以访问那个对象的synchronized代码（代码 块或方法）。通常，我们将使用this关键字引用执行该方法的对象。   
synchronized (this) {  
}
  
synchronized()保护代码块时，如果使用对象作为参数，而不是this
JVM可以保证只有一个线程可以访问那个对象保护所有的代码块
如: 我们用一个对象来控制name1属性的访问。所以，在任意时刻，只有一个线程能修改该属性。用另一个对象来控制 name2属性的访问。所以，在任意时刻，只有一个线程能修改这个属性。  
但是可能有两个线程同时运行，一个修改 name1属性而另一个修改name2属性。

synchronized和下面方法一起使用:  
nitifyAll() 通知所有的wait线程返回  
nitify() 通知一个wait的线程返回  
wait() 会释放对象锁; 被notity后，还需要等待锁，才能返回

# 使用Lock同步代码块
1,接口Lock, 实现类ReentrantLock  
2,它允许以一种更灵活的方式来构建同步块。   
3,Lock 接口比synchronized关键字提供更多额外的功能。新功能之一是实现的tryLock()方法。这种方法试图获取锁的控制权并且如果它不能获取该锁，是因为其他线程在使用这个锁，它将返回这个锁。   
4,使用synchronized关键字，当线程A试图执行synchronized代码块，如果线程B正在执行它，那么线程A将**只能阻塞**直到线程B执行完synchronized代码块。使用Lock锁，你可以执行tryLock()方法，这个方法返回一个 Boolean值表示，是否有其他线程锁。代码可以更加灵活。  
5,当有多个读和一个写时，Lock接口允许读写操作分离。  
6,Lock接口比synchronized关键字提供更好的性能。  

ReentrantLock其它方法:  
  getOwner() 方法来返回线程的名字。  
   ReentrantLock类保护方法getQueuedThreads()， 返回正在等待执行临界区的线程list。  
   hasQueuedThreads():此方法返回 Boolean 值表明是否有线程在等待获取此锁  
   getQueueLength(): 此方法返回等待获取此锁的线程数量  
   isLocked(): 此方法返回 Boolean 值表明此锁是否为某个线程所拥有  
   isFair(): 此方法返回 Boolean 值表明锁的 fair 模式是否被激活  


# 读写锁
ReadWriteLock接口，实现类：ReentrantReadWriteLock类
读操作：是通过在 ReadWriteLock接口中声明的readLock()方法获取的。这个锁是实现Lock接口的一个对象，所以我们可以使用lock()， unlock() 和tryLock()方法。
写操作：是通过在ReadWriteLock接口中声明的writeLock()方法获取的。这个锁是实现Lock接 口的一个对象，所以我们可以使用lock()， unlock() 和tryLock()方法。

```
ReadWriteLock rwlock = new ReentrantReadWriteLock();
//读锁
rwlock.readLock().lock();
//读操作......
//释放读锁  
rwlock.readLock().unlock();

// 写锁
rwlock.writeLock().lock();
//写操作......
//释放写锁  
rwlock.writeLock().unlock();  

```



# 锁的公平性
在ReentrantLock类和 ReentrantReadWriteLock类的构造器中，允许一个名为fair的boolean类型参数，它允许你来控制这些类的行为。  
false值: 默认值，启用非公平模式。在这个模式中，当有多个线程正在等待一把锁（ReentrantLock或者 ReentrantReadWriteLock），这个锁必须选择它们中间的一个来获得进入临界区，选择任意一个是**没有任何标准**的。  
true值: 开启公平模式。在这个模式中，当有多个线程正在等待一把锁（ReentrantLock或者ReentrantReadWriteLock），这个锁必须选择它们 中间的一个来获得进入临界区，**它将选择等待时间最长的线程**。

# Condition
Condition接口提供一种机制，阻塞一个线程和唤醒一个被阻塞的线程。  
Lock接口中的newCondition()方法来创建。

```
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

当一个线程在一个condition上调用await()方法时，它将自动释放锁的控制，并进入Condition变量的等待队列。

当一个线程在一个condition上调用signal()或signallAll()方法，一个或者全部在这个condition上等待的线程将被唤醒，从await方法返回。**返回前必须获得锁**。  

```
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void conditionWait() throws InterruptedException {
        lock.lock();
        try {
            //....
            condition.await();//await方法后，线程将释放锁，并且将自己沉睡，等待唤醒
        }finally {
            lock.unlock();
        }
    }

    public void conditionSignal(){
        lock.lock();//上面的线程释放锁后，这里可以获得锁
        try {
            //...
            condition.signal();//调用Condition的signal方法，唤醒上面线程，但此时锁没有释放，线程1没有真正唤醒
        }finally {
            lock.unlock();//释放锁，线程1将被执行
        }
    }
```


Condition接口提供不同版本的await()方法，如下：  
1,await(long time, TimeUnit unit):这个线程将会一直睡眠直到：  
(1)它被中断  
(2)其他线程在这个condition上调用singal()或signalAll()方法  
(3)指定的时间已经过了  
(4)TimeUnit类是一个枚举类型如下的常量：
DAYS,HOURS, MICROSECONDS, MILLISECONDS, MINUTES, NANOSECONDS,SECONDS  
2,awaitUninterruptibly():这个线程将不会被中断，一直睡眠直到其他线程调用signal()或signalAll()方法  
3,awaitUntil(Date date):这个线程将会一直睡眠直到：  
(1)它被中断  
(2)其他线程在这个condition上调用singal()或signalAll()方法  
(3)指定的日期已经到了  

# 参考资料
并发中文网: http://ifeve.com/
