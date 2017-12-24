---
title: redis主从复制和集群搭建
date: 2017-12-24 17:39:24
update: 2017-12-24 17:39:24
categories: nosql
tags: [nosql,redis]
---
# Redis主从复制
配置从服务器:  
到redis安装目录
```
$cp redis.conf redis-s1.conf
```
更改redis-s1.conf, 进行下面改动

```
port 6380
slaveof 127.0.0.1 6379
```
slaveof后面就是主服务器的地址和端口。  
启动：
```
$redis-server redis-s1.conf
1174:S 24 Dec 15:57:59.619 * Connecting to MASTER 127.0.0.1:6379
1174:S 24 Dec 15:57:59.619 * MASTER <-> SLAVE sync started
1174:S 24 Dec 15:57:59.619 * Non blocking connect for SYNC fired the event.
1174:S 24 Dec 15:57:59.619 * Master replied to PING, replication can continue...
1174:S 24 Dec 15:57:59.620 * Trying a partial resynchronization (request 2f7bd6f4987a7a6b93f1faa4822311f85c98d3e2:1).
1174:S 24 Dec 15:57:59.620 * Full resync from master: 3cfd21b61e1304bb8c5194f5cbcece34080ed4e7:0
1174:S 24 Dec 15:57:59.620 * Discarding previously cached master state.
1174:S 24 Dec 15:57:59.639 * MASTER <-> SLAVE sync: receiving 773 bytes from master
1174:S 24 Dec 15:57:59.673 * MASTER <-> SLAVE sync: Flushing old data
1174:S 24 Dec 15:57:59.673 * MASTER <-> SLAVE sync: Loading DB in memory
1174:S 24 Dec 15:57:59.673 * MASTER <-> SLAVE sync: Finished with success
```
测试一下，连接到从服务器，看数据有没有同步过来。

```
localhost:~ yuchenghong$ redis-cli -h 127.0.0.1 -p 6380
127.0.0.1:6380> hvals user:zhangsan
1) "zhangsan"
2) "123456"
```
# Redis集群搭建
参考官方文档: https://redis.io/topics/cluster-tutorial  
要让集群正常运作至少需要三个主节点, 建议使用六个节点： 其中三个为主节点， 而其余三个则是各个主节点的从节点。  

集群需要ruby，先安装下ruby  
然后执行：
```
$sudo gem install redis
```

我这儿测试，是在同一台机器，所以需要有6个不同端口。  
进入一个新目录， 并创建六个以端口号为名字的子目录
```
localhost:redis-4.0.0 yuchenghong$ mkdir cluster-test
localhost:redis-4.0.0 yuchenghong$ cd cluster-test
localhost:cluster-test yuchenghong$ mkdir 7000 7001 7002 7003 7004 7005
```
复制redis.conf到每一个目录，并更改配置，把端口号改成和目录名一致  
```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```
cluster-enabled 选项用于开实例的集群模式;  
cluster-conf-file 选项则设定了保存节点配置文件的路径，默认值为 nodes.conf. 节点配置文件无须人为修改，它由Redis集群在启动时创建，并在有需要时自动进行更新。

  
到每一个目录下启动redis
```
localhost:cluster-test yuchenghong$ cd 7000
localhost:7000 yuchenghong$ redis-server redis.conf
```
因为 nodes.conf 文件不存在， 所以每个节点都为它自身指定了一个新的ID.实例会一直使用同一个 ID ，从而在集群中保持一个独一无二的名字。
```
1299:M 24 Dec 17:14:22.348 * No cluster configuration found, I'm 78c3e6402e7a7cf83f3ebb3cc4737fc386a2104a
```

现在我们已经有了六个正在运行中的 Redis 实例， 接下来我们需要使用这些实例来创建集群.  
到src目录下面，启动集群，redis-trib.rb是一个ruby程序。  

```
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: 2da67f756f9b3a5f5381c5a2fe790aa76898a12d 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 78c3e6402e7a7cf83f3ebb3cc4737fc386a2104a 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: a79979813b191a9b922907e72aca881d8b33339a 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 03ce0a0d976199f04cb6ec0fbae8aa0a2c435a04 127.0.0.1:7003
   replicates 2da67f756f9b3a5f5381c5a2fe790aa76898a12d
S: b5ec9455653ba3b61488e96a4fa3de3d51ac89b3 127.0.0.1:7004
   replicates 78c3e6402e7a7cf83f3ebb3cc4737fc386a2104a
S: da5354db107279333802c87b8abf52d65ad4b179 127.0.0.1:7005
   replicates a79979813b191a9b922907e72aca881d8b33339a
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 2da67f756f9b3a5f5381c5a2fe790aa76898a12d 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: da5354db107279333802c87b8abf52d65ad4b179 127.0.0.1:7005
   slots: (0 slots) slave
   replicates a79979813b191a9b922907e72aca881d8b33339a
M: 78c3e6402e7a7cf83f3ebb3cc4737fc386a2104a 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
M: a79979813b191a9b922907e72aca881d8b33339a 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: b5ec9455653ba3b61488e96a4fa3de3d51ac89b3 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 78c3e6402e7a7cf83f3ebb3cc4737fc386a2104a
S: 03ce0a0d976199f04cb6ec0fbae8aa0a2c435a04 127.0.0.1:7003
   slots: (0 slots) slave
   replicates 2da67f756f9b3a5f5381c5a2fe790aa76898a12d
[OK] All nodes agree about slots configuration.
```
成功了。  

连接redis集群测试一下：  
```
localhost:src yuchenghong$ redis-cli -h 127.0.0.1 -c -p 7000
127.0.0.1:7000> set name helloworld
-> Redirected to slot [5798] located at 127.0.0.1:7001
OK
127.0.0.1:7001> get name
"helloworld"

localhost:src yuchenghong$ redis-cli -h 127.0.0.1 -c -p 7001
127.0.0.1:7001> get name
"helloworld"
127.0.0.1:7001> set hello helloredis
-> Redirected to slot [866] located at 127.0.0.1:7000
OK


localhost:src yuchenghong$ redis-cli -h 127.0.0.1 -c -p 7003
127.0.0.1:7003> set hh nn
-> Redirected to slot [12077] located at 127.0.0.1:7002
OK
127.0.0.1:7002> get nn
-> Redirected to slot [9549] located at 127.0.0.1:7001
(nil)
127.0.0.1:7001> get name
"helloworld"
```
搞定，如果是不同的机器，原理差不多。配置好IP和端口就好了。 
