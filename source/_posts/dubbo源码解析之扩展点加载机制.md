---
title: dubbo源码解析之扩展点加载机制
date: 2018-01-28 00:10:05
update: 2018-01-28 00:10:05
categories: RPC
tags: [RPC]
---
从这节开始，真正的开始分析源码了。  

## 一、从Hello World开始
从最简单的例子入手逐步分析：  
定义下面的接口和实现类
```
public interface DemoService {

    String sayHello(String name);

}

public class DemoServiceImpl implements DemoService {

    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response form provider: " + RpcContext.getContext().getLocalAddress();
    }

}
```
定义spring.xml配置文件
```
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="demo-provider"/>

    <!-- use multicast registry center to export service -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>

    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- service implementation, as same as regular local bean -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- declare the service interface to be exported -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```
定义启动类

```
public class Provider {

    public static void main(String[] args) throws Exception {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();

        System.in.read(); // press any key to exit
    }

}
```
首先启动zookepper, 然后启动上面的Provider的main方法。  
用过的同学都知道，一个简单的dubbo服务就发布了。  

我们看下加载流程：  
1，从Provider的main方法开始，这个方法直接启动spring容器；  
2，spring容器加载dubbo-demo-provider.xml定义的dubbo标签，在上一节讲过，dubbo标签的解析类为：  
DubboNamespaceHandler和DubboBeanDefinitionParser  
3，加载dubbo标签中，有个service标签，其对应的处理类为ServiceBean。这个是最重要的类。  
ServiceBean实现了ApplicationListener接口的onApplicationEvent方法监听service标签的初始化。  
如下面的标签会被解析成ServiceBean，其初始化时会触发onApplicationEvent方法。  
```
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
```
ServiceBean的onApplicationEvent方法如下：
```
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}
```
4，ServiceBean继承ServiceConfig类，上面export()方法实际上最终会调用到ServiceConfig类doExportUrlsFor1Protocol()方法，这个方法就是实现服务的发布。  
5, 而ServiceConfig里面有静态变量:
```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
这个在调用到ServiceConfig方法时。需要首先初始化。因此，首先会加载第一个扩展点Protocol.class。  
接下来，ServiceConfig发布服务时，后陆续的加载所有需要加载的扩展。  
后面我们会逐步讲解。  

## 二、主要的注解
在开始讲ExtensionLoader之前，要说一下dubbo扩展机制中主要的注解。  

### SPI
标注一个接口为扩展接口。后面带参数表示默认实现，比如@SPI("dubbo")，其默认实现为"dubbo"。   
所有扩展必须加上SPI注解才能通过ExtensionLoader加载。  

### Activate
对于集合类扩展点，比如：Filter, InvokerListener, ExportListener, TelnetHandler, StatusChecker。  
可以同时加载多个实现，此时，可以用自动激活来简化配置：  

```
@Activate // 无条件自动激活
public class XxxFilter implements Filter {
    // ...
}
```

```
List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
```

### Adaptive: 扩展点自适应
可以注解到类或者方法。  

@Adaptive到类时，类会被加载为扩展点自适应(AdaptiveExtension), 这个类只能有一个。  
作用是从所有的扩展实现中,根据当前的url配置策略,选出指定的的扩展实现类执行。   
如：  
```
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory
```

@Adaptive到方法时。  
如果配置文件下的类都没有@Adaptive注解，则调用createAdatpiveExtensionClass去生成适配扩展工厂类的代码，并编译加载到内存。  
此时接口必须至少要有一个打上@Adaptive的方法，并且打上了@Adaptive注解的方法参数必须有URL类型参数或者有参数中存在getURL()方法。  
以protocal为例，看下生成的代码.  
```
@SPI("dubbo")
public interface Protocol {
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    
}
```
调用没有打上@Adaptive的方法时会抛出异常。  
```
package com.alibaba.dubbo.rpc;  
import com.alibaba.dubbo.common.extension.ExtensionLoader;  

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {  
       public void destroy() {  
           throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");  
       }  
       public int getDefaultPort() {  
           throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");  
       }  
       public com.alibaba.dubbo.rpc.Exporter (com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {  
          if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");  
          if (arg0.getUrl() == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");com.alibaba.dubbo.common.URL url = arg0.getUrl();  
          String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );  
         if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");  
          com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);  
           return extension.export(arg0);  
      }  
     public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {  
           if (arg1 == null) throw new IllegalArgumentException("url == null");  
          com.alibaba.dubbo.common.URL url = arg1;  
         String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );  
           if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");  
           com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);  
          return extension.refer(arg0, arg1);  
       }  
   }
```

## 三、ExtensionLoader
spi的具体约定为:当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要在代码里指定。  
具体到dubbo, dubbo的ExtensionLoader会加载META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal/这三个路径配置的类。  

dubbo基础类ExtensionLoader和URL贯穿整个框架，掌握这两个类的思想和源码，就相当于对dubbo有了大致的了解。  

ExtensionLoader位于dubbo-common工程的com.alibaba.dubbo.common.extension包。  

所有的扩展都保存在ExtensionLoader的全局变量中。对于一个扩展点只会加载一次，生成一个ExtensionLoader。  
```
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```
通过ExtensionLoader.getExtensionLoader(class type)传入一个SPI扩展返回一个ExtensionLoader实例。

终于到主题了。  
以上面说的Protocol为例，Protocol是尝试加载的第一个扩展。  
入口在ServiceConfig类的，静态变量中：  
```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

ExtensionLoader.getExtensionLoader(class type)这个方法就是获取扩展点的入口。  

```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    //type只能是接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    //必须要@SPI注解的接口
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    //一个type类型对应一个ExtensionLoader实例，并缓存
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        //如果在缓存中不存在，则创建实例，并缓存。
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```
接着看ExtensionLoader的私有构造函数，创建ExtensionLoader实例：

```
private ExtensionLoader(Class<?> type) {
    this.type = type;
    //如果扩展类型是ExtensionFactory,那么则设置为null
    //这里通过getAdaptiveExtension方法获取一个运行时自适应的扩展类型
    //每个Extension只能有一个@Adaptive类型的实现，如果没有dubbo会动态生成一个类
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
这个方法设置了objectFactory属性。他是一个ExtensionFactory类型，ExtensionFactory主要用于加载扩展的实现。  
1，如果传入的参数是ExtensionFactory.class，objectFactory会被设置为null,很容易理解了，如果本身就是ExtensionFactory，则不需要设置objectFactory。  
2, 如果参数不是ExtensionFactory.class，则获取ExtensionFactory.class的AdaptiveExtension。  

这里，我们最开始传进来的type参数是Protocol，而设置objectFactory又需要加载ExtensionFactory。  
因此实际上ExtensionFactory会在Protocol之前加载。  
到这里，Protocol扩展的ExtensionLoader对象就获取到了。  

接下来看ExtensionLoader对象的getAdaptiveExtension()方法。  
在获取到ExtensionLoader对象后，通过getAdaptiveExtension()方法，尝试从cachedAdaptiveInstance获取自适应扩展(AdaptiveExtension)实例。  
如果不存在，则创建，然后存到cachedAdaptiveInstance。  
```
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```
如果cachedAdaptiveInstance为空，则调用createAdaptiveExtension()方法获取实例。  

```
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```
获取实例前，getAdaptiveExtensionClass()方法得到Adaptive的class。  
```
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```
调用getExtensionClasses去加载之前说的META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal/三个目录下，去找类型对应的配置文件，这个配置文件里面的类如果有@Adaptive注解，则加载并直接赋值给cacheAdaptiveClass静态变量；  
如果配置文件下的类都没有@Adaptive注解，则调用createAdatpiveExtensionClass去生成适配扩展工厂类的代码，并编译加载到内存。 
前面提到的Protocol$Adpative类，就是生产的代码。  

获取到AdaptiveExtension的实例后，injectExtension(T instance)方法，把真正的实现类通过set的方法注入到AdaptiveExtension的实例中。  
这样就可以通过AdaptiveExtension实例来调用真实的实现。  

```
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //上面提到过objectFactory获取扩展的实现
                        //SpiExtensionFactory识别pt，也就是类型
                        //SpringExtensionFactory识别property，也就是名称
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
到此，AdaptiveExtension获取完毕。  

接下来看从ExtensionLoader对象中，根据名称获取当前扩展的指定实现。  

```
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```
尝试从变量中获取制定的实例，如果不存在，则调用createExtension(String name) 方法创建实例。  
创建实例的之前，很关键的是通过loadExtensionClasses()方法获取到class。  
```
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if (value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```
上面的loadFile就是加载META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal/这三个路径配置的类。  
剩下的细节可以不用再讲了，整个流程都已经很清楚了。  




