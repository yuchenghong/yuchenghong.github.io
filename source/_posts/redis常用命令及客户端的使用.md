---
title: redis常用命令及客户端的使用
date: 2017-12-24 15:32:39
update: 2017-12-24 15:32:39
categories: nosql
tags: [nosql,redis]
---
# 数据类型
Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

# Redis命令
执行redis命令必须先要启动客户端。  

```
#连接本地
redis-cli
#连接远程，加上Ip和密码参数。
redis-cli -h 127.0.0.1 -p 6379 -a "password"
```

## Redis key说明
可以用任何二进制的序列作为Key, 比如"key",或者二进制图片的内容，都能作为key。  
比较好的习惯是通过冒号:，把Key分成几段。如  
user:zhangsan:name


## 字符串

```
127.0.0.1:6379> set myname redis
OK
127.0.0.1:6379> get myname
"redis"
127.0.0.1:6379>
```
设置多个值
```
127.0.0.1:6379> mset user:zhangsan:name zhangsan user:zhangsan:password 123456
OK
127.0.0.1:6379> mget user:zhangsan:name user:zhangsan:password
1) "zhangsan"
2) "123456"
```

## Hash
存储键值对

```
127.0.0.1:6379> hmset user:zhangsan name "zhangsan" password "123456"
OK
127.0.0.1:6379> hvals user:zhangsan
1) "zhangsan"
2) "123456"
```
得到所有的Key
```
127.0.0.1:6379> hkeys user:zhangsan
1) "name"
2) "password"
```
得到单个值

```
127.0.0.1:6379> hget user:zhangsan password
"123456"
```

## List
列表包含多个有序值，既可以作为队列（先进先出），也可以作为栈（后进先出）  
LRANGE获取元素，指定第一个位置和最后一个位置，负数表示到末尾。
```
127.0.0.1:6379> rpush zhangsan:weblist www.baidu.com www.163.com www.google.com
(integer) 3
127.0.0.1:6379> lrange zhangsan:weblist 0 -1
1) "www.baidu.com"
2) "www.163.com"
3) "www.google.com"
```
删除指定的值, 数字代表删除多少个，为0时表示删除所有。

```
127.0.0.1:6379> lrem zhangsan:weblist 0 www.163.com
(integer) 1
127.0.0.1:6379> lrange zhangsan:weblist 0 -1
1) "www.baidu.com"
2) "www.google.com"
```
像队列一样获取第一个值弹出，lpop从左边，rpop从右边

```
127.0.0.1:6379> lpop zhangsan:weblist
"www.baidu.com"
127.0.0.1:6379> lrange zhangsan:weblist 0 -1
1) "www.google.com"
```
像队列一样压入值，lpush左边，rpush右边

```
127.0.0.1:6379> lpush zhangsan:weblist www.sohu.com
(integer) 2
127.0.0.1:6379> lrange zhangsan:weblist 0 -1
1) "www.sohu.com"
2) "www.google.com"
```
阻塞list, 阻塞知道有值可以取到， 后面的参数表示等待的秒数，我这里例子是等待1分钟

```
127.0.0.1:6379> brpop zhangsan:book 60
```

## Set
无序，没有重复值
```
127.0.0.1:6379> sadd zhangsan:book h1 h2 h3 h4 h5
(integer) 5
127.0.0.1:6379> smembers zhangsan:book
1) "h4"
2) "h3"
3) "h2"
4) "h1"
5) "h5"
```
取得两个的交集：sinter

```
127.0.0.1:6379> sadd lisi:book h2 h3 h6 h7
(integer) 2
127.0.0.1:6379> smembers lisi:book
1) "h3"
2) "h2"
3) "h6"
4) "h7"
127.0.0.1:6379> sinter zhangsan:book lisi:book
1) "h3"
2) "h2"
```
找出不同的值

```
127.0.0.1:6379> sdiff zhangsan:book lisi:book
1) "h4"
2) "h5"
3) "h1"
```

## 有序set
string 类型的集合，不允许有重复的元素。  
每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。  
```
127.0.0.1:6379> zadd sites 100 baidu 50 sohu 900 google
(integer) 3
127.0.0.1:6379> zadd sites 1 pptv
(integer) 1
127.0.0.1:6379> zadd sites 3 letv
(integer) 1

127.0.0.1:6379> zrange sites 0 -1
1) "pptv"
2) "letv"
3) "sohu"
4) "baidu"
5) "google"
```
找出某一分数范围的数据

```
127.0.0.1:6379> zrangebyscore sites 1 100
1) "pptv"
2) "letv"
3) "sohu"
4) "baidu"
```
# 设置数据到期
标记一个键为到期，EXPIRE命令  
ttl查询还剩余的时间
```
127.0.0.1:6379> set city beijing
OK
127.0.0.1:6379> expire city 20
(integer) 1
127.0.0.1:6379> ttl city
(integer) 17
```

SETEX直接创建一个有期限的string  

```
127.0.0.1:6379> setex city 20 beijing
OK
127.0.0.1:6379> ttl city
(integer) 7
```
更多命令参考:http://www.redis.cn/commands.html 

# Java代码操作Redis
Java使用Jedis包来操作redis  
引入jedis：
```
<dependency> 
    <groupId>redis.clients</groupId> 
    <artifactId>jedis</artifactId> 
    <version>2.9.0</version> 
</dependency>
```
示例

```
import redis.clients.jedis.Jedis;
 
public class RedisTest {
    public static void main(String[] args) {
        //连接本地的 Redis 服务
        Jedis jedis = new Jedis("localhost");
        System.out.println("连接成功");
        //字符串
        jedis.set("runoobkey", "www.runoob.com");
        // 获取存储的数据并输出
        System.out.println("redis 存储的字符串为: "+ jedis.get("runoobkey"));
        
        //list
        jedis.lpush("site-list", "Runoob");
        jedis.lpush("site-list", "Google");
        jedis.lpush("site-list", "Taobao");
        // 获取存储的数据并输出
        List<String> list = jedis.lrange("site-list", 0 ,2);
        for(int i=0; i<list.size(); i++) {
            System.out.println("列表项为: "+list.get(i));
        }
    }
}
```

