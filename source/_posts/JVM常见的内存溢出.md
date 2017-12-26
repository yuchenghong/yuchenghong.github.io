---
title: JVM常见的内存溢出
date: 2017-12-26 22:19:54
update: 2017-12-26 22:19:54
categories: jvm
tags: [jvm,java]
---
那些区域会发生内存溢出？  
除了程序计数器，其他几块运行时数据区都有可能发生内存溢出。  

# 堆溢出
报错：java.lang.OutOfMemoryError: Java heap space  
碰到这个问题，从两个方面入手：  
1，判断是否是内存泄露。  
通过内存分析工具对堆快照进行分析。如何分析请看本文的[Dump分析内存]章节，有详细的说明。 
2，如果不是内存泄露。  
检查堆参数：-Xmn -Xms -Xmx。  

# 栈溢出
报错：java.lang.StackOverflowError : Thread Stack space  
或者类似java.lang.OutOfMemoryError: unable to create new native thread 
1，如果调用构造函数的 “层”太多了，以致于把栈区溢出了。会抛出StackOverflowError异常。  
2，如果虚拟机在扩展时，无法申请到足够的内存，则抛出OutOfMemoryError异常。  

解决办法：  
1，检查程序，函数层次 还有 创建的线程数量。  
2，通过-Xss: 来设置每个线程的Stack大小。

# 方法区溢出-java8以前
报错：java.lang.OutOfMemoryError: PermGen space  
这块内存主要是被JVM存放Class，GC不会在主程序运行期对PermGen space进行清理，所以如果你的系统会载入很多CLASS的话，就很可能出现PermGen space溢出。一般发生在程序的启动阶段。  

解决方法： 
通过-XX:PermSize和-XX:MaxPermSize设置永久代大小即可。

# 元空间溢出-Java8以后
报错：java.lang.OutOfMemoryError: Metaspace
默认元空间只受本地物理内存的限制。  
如果这块有溢出，从两个方面检查：  
1，检查-XX:MaxMetaspaceSize参数，是否限制了元空间大小，如果是。则考虑调大或者直接取消限制。  
2，如果是物理内存溢出，则考虑加大物理内存。  

# 本机直接内存溢出
报错：java.lang.OutOfMemoryError
检查dump文件，如果没有明显异常，就要怀疑是不是直接内存溢出了。  
一般的原因是由于NIO直接分配内存导致的。  

解决办法：  
检查代码。

# Dump分析内存
采用jmap命令导出内存转储快照Dump文件。  
比如tomcat, 首先查询到你对应的Tomcat的pid：

```
ps -aux|grep xxx-tomcat
```
比如得到的pid: 9809, 然后执行命令，其中dump001.hprof为导出的文件名：
```
jmap -dump:format=b,file=dump001.hprof 9809
```
导出完毕。下载先来，有eclipse打开，打开前需要安装一下插件(baidu找下插件)。  
打开文件后，可以看到那些对象占的内存比较多。来排查代码问题。  

