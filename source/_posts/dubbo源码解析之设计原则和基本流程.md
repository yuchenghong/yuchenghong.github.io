---
title: dubbo源码解析之设计原则和基本流程
date: 2018-01-27 15:14:39
update: 2018-01-27 15:14:39
categories: RPC
tags: [RPC]
---
源码阅读的过程是个很痛苦的过程。  
经常debug跟踪代码，绕了不知道多少弯，最终跟丢。不知道身在何处。  
合上电脑，不知道自己是谁，为什么来到这世界，要去到哪里。。。  

把源码解析过程写出来，困难也超出想象，我构思了一周多。   
感觉知识真是不够用。很多能意会不能言传的逻辑，很是苦恼。  
不管怎样，尝试写一写，理清思路，希望所写的，读者都能很容易理解。  

开始看dubbo。  
本文的内容只是把大致的设计原理和流程介绍一下，每个部分详细的源码分析，后面会一一的介绍。  
在阅读源码之前，一定要把官方设计文档仔细读一下：http://dubbo.io/books/dubbo-dev-book/  

## 一、设计原则
先看看官方对dubbo设计的说明：  

>dubbo采用 Microkernel + Plugin 模式(微核心+插件式)，
Microkernel 只负责组装 Plugin，Dubbo 自身的功能也是通过扩展点实现的，
也就是 Dubbo 的所有功能点都可被用户自定义扩展所替换。  
采用 URL 作为配置信息的统一格式，所有扩展点都通过传递 URL 携带配置信息。  

>dubbo采用的是 JDK 标准的 SPI 扩展机制，参见：java.util.ServiceLoader。  

>dubbo 改进了 JDK 标准的 SPI 的以下问题：  
1,JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。  
2,如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，
但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，
和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。  
3,增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。  

>在扩展类的jar包内，放置扩展点配置文件 META-INF/dubbo/接口全限定名，内容为：配置名=扩展实现类全限定名，多个实现类用换行符分隔。  
如：  
以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：META-INF/dubbo/com.alibaba.dubbo.rpc.Protocol，内容为： 
xxx=com.alibaba.xxx.XxxProtocol

dubbo通过ExtensionLoader类来加载SPI扩展类，然后所有的配置都转换成URL对象实例。  
掌握了ExtensionLoader类，就算是掌握了dubbo的核心实现。  

## 二、spring标签扩展
用过dubbo的同学都知道，dubbo服务的配置信息是放到spring的配置里面的，类似下面：
```
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
```
dubbo标签并不是spring的标签，这个是如何解析的呢？  

Spring提供了可扩展Schema的支持，完成一个自定义配置一般需要以下步骤：  
1，编写JavaBean  
2，编写XSD文件  
3，编写NamespaceHandler和BeanDefinitionParser完成解析工作  
4，编写spring.handlers和spring.schemas  
详细可以参考spring源码关于schema扩展部分。  

以dubbo:service标签为例：  
1,首先定义bean: com.alibaba.dubbo.config.spring.ServiceBean  
2,定义xsd：dubbo-config-spring项目resource/META-INF/dubbo.xsd.  
部分定义：
```
<xsd:element name="service" type="serviceType"> 
        <xsd:annotation> 
            <xsd:documentation><![CDATA[ Export service config ]]></xsd:documentation> 
        </xsd:annotation> 
</xsd:element>
```
3, 编写DubboNamespaceHandler和DubboBeanDefinitionParser对dubbo标签进行解析。  
4, 编写spring.handlers和spring.schemas：dubbo-config-spring项目resource/META-INF目录下：  
spring.handlers文件的内容：
```
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```
以上表示当使用到名为"http://code.alibabatech.com/schema/dubbo"的schema引用时，会通过com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler 来完成解析。  

spring.schemas 文件的内容：
```
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
```

所以。  
Spring 在遇到 dubbo 名称空间时，会回调 DubboNamespaceHandler。  
所有dubbo的标签，都统一用 DubboBeanDefinitionParser 进行解析，基于一对一属性映射，将 XML 标签解析为 Bean 对象。  
DubboNamespaceHandler源码如下：  
```
public void init() {
    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
    registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
    registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
    registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
    registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
    registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
    registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
    registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
    registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
    registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
}
```

## 三、dubbo启动过程
dubbo不依赖任何web容器，因此，可以直接用一个main方法启动：
```
com.alibaba.dubbo.container.Main.main
```
这个方法会去获取Container.class的扩展，默认实现是SpringContainer。  
最终会通过ApplicationContext的start方法启动spring容器，来实现dubbo服务的启动。  
```
public void start() {
        String configPath = ConfigUtils.getProperty(SPRING_CONFIG);
        if (configPath == null || configPath.length() == 0) {
            configPath = DEFAULT_SPRING_CONFIG;
        }
        context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));
        context.start();
    }
```

有时候，我们可以通过把dubbo服务的xml注册到tomcat的web.xml，通过web服务器来加载dubbo服务。  
原理是一样的，只是这样就跳过了Container.class的加载，直接启动了。  

如果是新手，第一次看到这里，可能不能完全理解。  
不过没关系，随着后面不断的深入介绍，这个自然就会明白。  
现在只需记得两点，一、dubbo默认使用spring容器来启动加载服务，二、加载服务也是通过扩展Container.class来实现。  


