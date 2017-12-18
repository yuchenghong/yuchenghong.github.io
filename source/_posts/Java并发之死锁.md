---
title: Java并发之死锁
date: 2017-12-18 22:48:58
update: 2017-12-18 22:48:58
categories: java
tags: [java]
---

简单聊一下死锁。  
当一个线程永远的持有一个锁，并且其他线程都尝试获得这个锁，就会发生死锁。  
如果发生死锁，通常JVM没有办法自动恢复，只能重启。对线上系统来说，这是灾难性的。
# 死锁的发生场景、例子  
1，最简单的例子: 
线程A持有锁1并尝试获得锁2，线程B占有锁2并尝试获得锁1。两个线程将会永远的等待下去。  

2，多个线程，持有锁的同时，尝试获得对方的锁，形成一个复杂的环状依赖。这些线程都将无限的等待。  
  
3，两个线程以不同的顺序获得相同的锁  
举个栗子：  
一段转账的代码，从A账号转钱到B账号，为了保持数据一致性，操作前需要对着两个账号加锁，避免账号被同步更改而产生错误的数据。

```
public void transfer(Account fromAccount,Account toAccount){
    synchronized(fromAccount){
        synchronized(toAccount){
            //转账动作. 
        }
    }
}
```
上面这段代码，是什么情况下会死锁呢？  
这样一个场景：  
A线程： 
```
transfer(account1,account2)
```

B线程： 
```
transfer(account2,account1)
```
A获得account1的锁，等待account2的锁； B获得account2的锁，等待account1的锁。 两个线程无限的等待。   
这就是“两个线程以不同的顺序获得相同的锁”。  


# 死锁产生的条件  
死锁的发生必须满足4个条件：  
1，资源是不能共享的，同一资源不能被不同的线程占有  
2，任务持有一个资源的同时，等待别的任务的资源  
3，资源不能抢占，不能强行从其他线程抢占  
4，产生循环等待  
  
要发生死锁，上面的条件缺一不可。 
# 如何避免死锁  
上面说到了死锁的发生必须要满足四个条件。  
所以避免死锁，我们可以从这四点着手，打破其中一条即可。  
  
在多线程下，可以从几个方面：  
1，加锁顺序：对于同样的资源，其加锁顺序一样。  
比如上面的转账例子，

```
transfer(account1,account2)
transfer(account2,account1)
```
这两个调用，不管参数的顺序如何，在加锁时，保证account1,account2是同样的顺序，就一定不会有死锁。  
具体实现，可以设计一个哈希算法，来保证加锁的顺序。代码如下，这样对于同样的账号，不管是在哪个参数传进来，其hash值大小固定，加锁顺序就一定是一样的。 **问题完美解决。**  

```
public void transfer(Account fromAccount,Account toAccount){
    int fromHash = hash(fromAccount);
    int toHash = hash(toAccount);
    
    if(fromHash>toHash){
        synchronized(fromAccount){
            synchronized(toAccount){
                //转账动作. 
            }
        }
    }else{
        synchronized(toAccount){
            synchronized(fromAccount){
                //转账动作. 
            }
        }
    }
}
```
2，加锁时间  
使用Lock对象来加锁， 在尝试获取锁的时候，加上时间参数，超过时间就放弃操作，避免永久等待。  



