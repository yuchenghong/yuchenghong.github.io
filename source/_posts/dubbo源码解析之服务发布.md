---
title: dubbo源码解析之服务发布
date: 2018-02-03 17:56:14
update: 2018-02-03 17:56:14
categories: RPC
tags: [RPC]
---
开始之前，仔细看看官方文档:http://dubbo.io/books/dubbo-dev-book/design.html  
官方文档已经把整体设计和调用过程大致都介绍过了。  

# 扩展点加载顺序
dubbo自身的功能都是通过扩展点实现的。  
首先看看，dubbo服务发布过程需要用到哪些扩展，可以通过打印日志查看到扩展的加载，下面是清单：  

- com.alibaba.dubbo.rpc.Protocol  
RPC 协议扩展，封装远程调用细节。
- com.alibaba.dubbo.common.extension.ExtensionFactory  
扩展点本身的加载容器，可从不同容器加载扩展点。
- com.alibaba.dubbo.common.compiler.Compiler  
Java 代码编译器，用于动态生成字节码，加速调用。
- com.alibaba.dubbo.rpc.ProxyFactory  
动态代理扩展，将 Invoker 接口转换成业务接口。
- com.alibaba.dubbo.registry.RegistryFactory
负责服务的注册与发现。
- com.alibaba.dubbo.remoting.telnet.TelnetHandler
Telnet 命令扩展，所有服务器均支持 telnet 访问，用于人工干预。
- com.alibaba.dubbo.rpc.cluster.ConfiguratorFactory  
向注册中心修改注册信息
- com.alibaba.dubbo.rpc.Filter  
服务提供方和服务消费方调用过程拦截，Dubbo 本身的大多功能均基于此扩展点实现，每次远程方法执行，该拦截都会被执行
- com.alibaba.dubbo.cache.CacheFactory  
用请求参数作为 key，缓存返回结果。
- com.alibaba.dubbo.monitor.MonitorFactory  
负责服务调用次和调用时间的监控。
- com.alibaba.dubbo.validation.Validation  
参数验证扩展点。
- com.alibaba.dubbo.rpc.ExporterListener  
当有服务暴露时，触发该事件。
- com.alibaba.dubbo.rpc.cluster.Cluster  
当有多个服务提供方时，将多个服务提供方组织成一个集群，并伪装成一个提供方。
- com.alibaba.dubbo.remoting.Transporter  
远程通讯的服务器及客户端传输实现。
- com.alibaba.dubbo.remoting.exchange.Exchanger  
基于传输层之上，实现 Request-Response 信息交换语义。
- com.alibaba.dubbo.remoting.Dispatcher  
通道信息派发器，用于指定线程池模型。
- com.alibaba.dubbo.common.threadpool.ThreadPool  
服务提供方线程程实现策略，当服务器收到一个请求时，需要在线程池中创建一个线程去执行服务提供方业务逻辑。
- com.alibaba.dubbo.remoting.Codec2  
Dubbo的远程调用需要对传输的数据进行编码解码，dubbo的Codec2接口定义了编码解码规范

# 重要的接口和类介绍
补充一下服务发布过程中用到的很重要的接口和类：  
- Invoker,一个可执行对象，能够根据方法名称、参数得到相应的执行结果。
- Invocation,包含了需要执行的方法、参数等信息
- Exporter,负责维护invoker的生命周期，包含一个Invoker对象
- ProtocolFilterWrapper将所有 Filter 组装成链，在链的最后一节调用真实的引用。
- ProtocolListenerWrapper将所有 InvokerListener 和 ExporterListener 组装集合，在暴露和引用前后，进行回调。
Dubbo的开源版本中没有监听的实现，但是开放了口子，业务方如有需要可以利用这个功能实现特定的业务。
- DubboProtocol.export()服务导出功能：
1. 创建一个DubboExporter，封装Invoker  
2. 根据Invoker的url获取ExchangeServer通信对象（负责与客户端的通信模块）
- ExchangeHandler该类是真正与客户端通信，在获取到Invocation参数后，调用getInvoker()来获取本地的Invoker（先从exporterMap中获取Exporter），就可以调用服务了
- TelnetHandler: telnet访问服务器，命令执行服务。
- Codec2:Dubbo的远程调用需要对传输的数据进行编码解码，dubbo的Codec2接口定义了编码解码规范

# 服务发布过程
从例子开始：
```
    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="demo-provider"/>

    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>

    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- service implementation, as same as regular local bean -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- declare the service interface to be exported -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
```

1,spring解析dubbo schema, 也就是上面的xml文件；
2,解析dubbo:service，初始化ServiceBean，而ServiceBean使用spring机制注册Linstner，当bean初始化时，调用export()方法

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

3,export()方法会调到ServiceConfig.doExportUrls()。  

```
private void doExportUrls() {
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```
loadRegistries(true) 方法会把注册服务封装成URL：  

```
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=4350&registry=zookeeper&timestamp=1517645217298
```
4,然后会调到ServiceConfig.doExportUrlsFor1Protocol()方法。
这个方法很长，下面是暴露服务的关键点。  
```
String scope = url.getParameter(Constants.SCOPE_KEY);
// don't export when none is configured
if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

    // export to local if the config is not remote (export to remote only when config is remote)
    //这里的url格式：dubbo://192.168.1.7:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.1.7&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4661&side=provider&timestamp=1517647878680
    if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
        exportLocal(url);
    }
    // export to remote if the config is not local (export to local only when config is local)
    if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
        if (logger.isInfoEnabled()) {
            logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
        }
        if (registryURLs != null && registryURLs.size() > 0) {
            for (URL registryURL : registryURLs) {
                url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                URL monitorUrl = loadMonitor(registryURL);
                if (monitorUrl != null) {
                    url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                }
                if (logger.isInfoEnabled()) {
                    logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                }
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        } else {
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
            DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

            Exporter<?> exporter = protocol.export(wrapperInvoker);
            exporters.add(exporter);
        }
    }
}
```
本地暴露方法：  

```
private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(LOCALHOST)
                .setPort(0);
        ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
    }
}
```
这里的proxyFactory会是默认的JavassistProxyFactory；  
最终的invoker URL为：  
injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.1.7&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4661&side=provider&timestamp=1517647878680


5,再看远程暴露方法：

```
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
```
proxyFactory.getInvoker()->实际是JavassistProxyFactory.getInvoker()  
获得的invoker URL为：  
registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo%3A%2F%2F192.168.1.7%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D192.168.1.7%26bind.port%3D20880%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D4661%26side%3Dprovider%26timestamp%3D1517647878680&pid=4661&registry=zookeeper&timestamp=1517647853389  


6,接着下一行

```
Exporter<?> exporter = protocol.export(wrapperInvoker);
```
这里的protocol就是Protocol$Adpative，代码生成的类和对象，会根据URL参数觉得使用哪个实例，这儿参数是registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService，因此实例应该为RegistryProtocol。

7,实际会调用RegistryProtocol.export()

```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    //启动netty，发布服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    URL registryUrl = getRegistryUrl(originInvoker);

    //registry provider
    //通过registryFactory获取一个Registry
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    boolean register = registedProviderUrl.getParameter("register", true);

    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

    if (register) {
        //ZookeeperRegistry.doRegister里真正注册
        register(registryUrl, registedProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    //ZookeeperRegistry.doSubscribe里真正订阅监听
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //Ensure that a new exporter instance is returned every time export
    return new Exporter<T>() {
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }

        public void unexport() {
            try {
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                registry.unregister(registedProviderUrl);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                overrideListeners.remove(overrideSubscribeUrl);
                registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    };
}
```
比较关键的：  
doLocalExport方法启动netty,发布服务；  
register方法，zookeeperRegistry注册；  
下面单独分析。  

8,先看doLocalExport(originInvoker)方法。  
方法里面最关键的代码：  
protocol.export(invokerDelegete), originInvoker);  
这里的protocol为DubboProtocol  
```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
    //这里网络通讯的入口，默认创建netty服务。
    openServer(url);
    optimizeSerialization(url);
    return exporter;
}
```

9,接着看openserver()方法，会调用createServer()

```
private ExchangeServer createServer(URL url) {
    // send readonly event when server closes, it's enabled by default
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    // enable heartbeat by default
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    ExchangeServer server;
    try {
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```
Exchangers.bind()段  
首先会调用下面的方法获取Exchanger的扩展实现，并返回。  
ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);  
返回的是HeaderExchanger对象，再调用HeaderExchanger.bind(url, handler)方法。  

10,HeaderExchanger.bind(url, handler) ->Transporters.bind()

```
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
```
ExtensionLoader获取Transporter.class扩展实现。  
默认会是NettyTransporter。  
Transporters.bind()实际会调用到NettyTransporter.bind(), 返回NettyServer对象。  

11,NettyServer的doOpen()方法就是开启一个netty服务器了。 详细就不讲了。  
```
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    bootstrap = new ServerBootstrap(channelFactory);

    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    channels = nettyHandler.getChannels();
    // https://issues.jboss.org/browse/NETTY-365
    // https://issues.jboss.org/browse/NETTY-379
    // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        public ChannelPipeline getPipeline() {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            /*int idleTimeout = getIdleTimeout();
            if (idleTimeout > 10000) {
                pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
            }*/
            pipeline.addLast("decoder", adapter.getDecoder());
            pipeline.addLast("encoder", adapter.getEncoder());
            pipeline.addLast("handler", nettyHandler);
            return pipeline;
        }
    });
    // bind
    channel = bootstrap.bind(getBindAddress());
}
```

到此为止，一个服务就创建好了。  

