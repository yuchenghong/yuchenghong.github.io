---
title: RPC原理和实现
date: 2018-01-21 13:29:10
update: 2018-01-21 13:29:10
categories: RPC
tags: [RPC]
---
筹划了好久，一直想仔细总结分析dubbo的源码。以前陆陆续续看过了很多dubbo源代码，但没有系统性的研究，也没有形成总结性的分享文档。    
今天从RPC入手，分析dubbo核心源码，漫长而枯燥的过程，希望能坚持下去。  

为什么要分析dubbo源码？  
首先，分析源码是最有效的学习方式，优秀的源码都是业界多年沉积的精华，非常值得我们学习；  
其次，当一个团队引进一项技术，如dubbo，就需要有一个人能对这个技术有非常深入的理解，至少阅读过这个项目的源码。  

回到正题，什么是RPC？  
RPC全程Remoto Porcess Call, 远程过程调用。将原来的本地调用转变为调用远程服务器上的方法，提升系统的处理能力。  
RPC是实现分布式服务的基础。  
随着互利网和分布式的发展，RPC的变得越来越重要。  

RPC实现包括服务提供者和服务消费者。  
服务消费者发送RPC请求给服务提供者，服务提供者根据调用者的参数执行请求方法，并将结果返回给服务消费者。  

下面代码实现简单的RPC。这段代码来自于dubbo作者梁飞的博客，通过这段代码，我们能很清晰的了解整个RPC的过程和原理。  

1，最核心的服务暴露和服务引用：  
暴露服务：服务提供方暴露服务，接受调用者的参数，执行请求方法，并返回结果；  
服务引用：服务消费者发送请求参数，把请求发给提供方，并获得提供方的返回结果。  
```
public class RpcFramework {
    /**
     * 暴露服务
     * 
     * @param service 服务实现
     * @param port 服务端口
     * @throws Exception
     */
    public static void export(final Object service, int port) throws Exception {
        if (service == null)
            throw new IllegalArgumentException("service instance == null");
        if (port <= 0 || port > 65535)
            throw new IllegalArgumentException("Invalid port " + port);
        System.out.println("Export service " + service.getClass().getName() + " on port " + port);
        ServerSocket server = new ServerSocket(port);
        for(;;) {
            try {
                final Socket socket = server.accept();
                new Thread(new Runnable() {
                    public void run() {
                        try {
                            try {
                                ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
                                try {
                                    String methodName = input.readUTF();
                                    Class<?>[] parameterTypes = (Class<?>[])input.readObject();
                                    Object[] arguments = (Object[])input.readObject();
                                    ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());
                                    try {
                                        Method method = service.getClass().getMethod(methodName, parameterTypes);
                                        Object result = method.invoke(service, arguments);
                                        output.writeObject(result);
                                    } catch (Throwable t) {
                                        output.writeObject(t);
                                    } finally {
                                        output.close();
                                    }
                                } finally {
                                    input.close();
                                }
                            } finally {
                                socket.close();
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }).start();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
    /**
     * 引用服务
     * 
     * @param <T> 接口泛型
     * @param interfaceClass 接口类型
     * @param host 服务器主机名
     * @param port 服务器端口
     * @return 远程服务
     * @throws Exception
     */
    @SuppressWarnings("unchecked")
    public static <T> T refer(final Class<T> interfaceClass, final String host, final int port) throws Exception {
        if (interfaceClass == null)
            throw new IllegalArgumentException("Interface class == null");
        if (! interfaceClass.isInterface())
            throw new IllegalArgumentException("The " + interfaceClass.getName() + " must be interface class!");
        if (host == null || host.length() == 0)
            throw new IllegalArgumentException("Host == null!");
        if (port <= 0 || port > 65535)
            throw new IllegalArgumentException("Invalid port " + port);
        System.out.println("Get remote service " + interfaceClass.getName() + " from server " + host + ":" + port);
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[] {interfaceClass}, (proxy, method, arguments) -> {
            Socket socket = new Socket(host, port);
            try {
                ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream());
                try {
                    output.writeUTF(method.getName());
                    output.writeObject(method.getParameterTypes());
                    output.writeObject(arguments);
                    ObjectInputStream input = new ObjectInputStream(socket.getInputStream());
                    try {
                        Object result = input.readObject();
                        if (result instanceof Throwable) {
                            throw (Throwable) result;
                        }
                        return result;
                    } finally {
                        input.close();
                    }
                } finally {
                    output.close();
                }
            } finally {
                socket.close();
            }
        });
    }
}
```

2，定义一个接口和实现类，模拟实际的业务。 

```
/**
 * HelloService
 * 
 * @author william.liangf
 */
public interface HelloService {

    String hello(String name);

}

/**
 * HelloServiceImpl
 * 
 * @author william.liangf
 */
public class HelloServiceImpl implements HelloService {

    public String hello(String name) {
        return "Hello " + name;
    }

}
```



3，服务提供者代码：  

```
/**
 * RpcProvider
 * 
 * @author william.liangf
 */
public class RpcProvider {

    public static void main(String[] args) throws Exception {
        HelloService service = new HelloServiceImpl();
        RpcFramework.export(service, 1234);
    }

}
```

4，服务消费者代码：

```
/**
 * RpcConsumer
 * 
 * @author william.liangf
 */
public class RpcConsumer {
    
    public static void main(String[] args) throws Exception {
        HelloService service = RpcFramework.refer(HelloService.class, "127.0.0.1", 1234);
        for (int i = 0; i < Integer.MAX_VALUE; i ++) {
            String hello = service.hello("World" + i);
            System.out.println(hello);
            Thread.sleep(1000);
        }
    }
    
}
```
5，运行：  
先启动RpcProvider，再启动RpcConsumer。  

最简单的RPC框架就搭建好了。