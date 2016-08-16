---
layout: post
title: Ele  SOA Container
categories: [blog]
tags: [Java, SOA]
description: 
---

Ele SOA Container

[TOC]

# Core

## Container

```java
public class Container {
    private final static Log logger = LogFactory.getLog(Container.class);

    public static void main(String[] args) {
        try {
            CoreBootstrap.getInstance().include((args != null && args.length > 0) ? args : new String[] { CoreConsts.DEFAULT_CONFIG_PATH }).start();
        } catch (Throwable t) {
            logger.error("Throwable Occurs in Container!", t);
            CoreBootstrap.getInstance().stop(-1);
        }
    }
}
```

即通过CoreBootstrap.getInstance().include(new SOAConf[]{this.soaConfig}).start()来启动，这种方式需要启动时将参数传入main()，需要按照约定。

如果以arena.api.melody为例：

```
applicationDefaultJvmArgs=["-DAPPID=arena.api.melody",
                           "-Dlistener=me.ele.napos.arena.api.melody.impl.ArenaApiMelodyContextListener",
                           "-DCONF_PATH=$rootDir/config",
                           "-Delog.config=$rootDir/config/elog.xml"]

```

## CoreBootstrap

CoreBootstrap提供了更加灵活的配置方法：

```java
   public CoreBootstrap include(SOAConf... soaConfs) {
        if (soaConfs != null) {
            for (SOAConf soaConf : soaConfs) {
                if (soaConf != null) {
                    include(soaConf.getCommonConf());
                    include(soaConf.getServerConf());
                    include(soaConf.getClientConfs());
                }
            }
        }
        return this;
    }
```

只需要将SOAConf实例传入即可。

## CoreInitializer

CoreBootstrap.start()通过CoreInitializer.init()启动。

CommonConf完成Huskar、MetricClient的启动和Etrace的URL配置。在RPC的InvokeHandler里会调用Trace，Trace本身完成了一个类似Statsd、Collectd的实现，包括Ejedis、Ejdbc都是通过Trace完成数据发送。

ps：Ejedis通过拼接字节码完成了替换了Jedis本省的几个类。Ejdbc在提交的时候调用Trace记录事件。

```java
 public void commit() throws SQLException {

        Transaction t = Trace.newTransaction(Constants.SQL, "commit");
        Trace.logEvent(Constants.SQL_DATABASE, dbName);

        try {
            if (CommentGenerator.isConnToDal(username)) {
                setComment(CommentGenerator.genAnnotation());
            }
            // business logic
            conn.commit();
            /////
            t.setStatus(Constants.SUCCESS);
        } catch (SQLException e) {
            Trace.logError(e);
            t.setStatus(e);
            throw e;
        } finally {
            t.complete();
            if (CommentGenerator.isConnToDal(username)) {
                setComment(null);// reset
            }
        }
    }
```

ServerConf完成三个server的启动：

startBizServer：业务的Server

startDaemonServer： 守护server，包括心跳、配置、dump等。

registerService： 服务注册，appspec.yml，即supervisor

## Server

### 业务Server

```java
    private static void startBizServer(ServerConf serverConf) {
        ServerProtocol serverProtocol = Enum.valueOf(ServerProtocol.class, serverConf.getProtocol());
        if (serverProtocol == null) {
            throw new CoreException("Unsupported server protocol(" + serverConf.getProtocol() + ")!");
        }
        server = serverProtocol.createServer(serverConf);
        server.start(serverConf.getPort());
    }
```

#### 安全的单例模式

```java
public enum ServerProtocol {
    json {
        @Override
        public Server<?> createServer(ServerConf serverConf) {
            return new JsonServer(serverConf.getName(), serverConf.getGroup(), serverConf.isValidatable(), createTaskExecutor(serverConf),
                    createDgStrategy(serverConf), createAuthStrategy(serverConf), serverConf.getInitializer(), serverConf.getInterfaces(),
                    serverConf.getWorkerGroupSize());
        }

        @Override
        protected DgStrategy createDgStrategy(ServerConf serverConf) {
            return new JsonHuskarDgStrategy(HuskarHandle.get().getMySwitch());
        }
    };

    public abstract Server<?> createServer(ServerConf serverConf);

    protected abstract DgStrategy createDgStrategy(ServerConf serverConf);

    protected TaskExecutor createTaskExecutor(ServerConf serverConf) {
        return new FIFRTaskExecutor(serverConf.getName(), serverConf.getThreadPoolSize(), serverConf.getBufferQueueSize() > 0 ? serverConf.getBufferQueueSize()
                : serverConf.getThreadPoolSize());
    }

    protected AuthStrategy createAuthStrategy(ServerConf serverConf) {
        HuskarAuthStrategy huskarAuthStrategy = new HuskarAuthStrategy();
        try {
            HuskarHandle.get().getConfigHandle().register(new DataIncidentRegistry<>(huskarAuthStrategy, serverConf.getName(), HuskarConsts.CLUSTER_OVERALL));
        } catch (HuskarException e) {
            throw new CoreException(e);
        }
        return new MergedAuthStrategy(huskarAuthStrategy,    							serverConf.getInitializer().getAuthStrategy());
    }
}
```

# RPC

#### JsonServer

```
public class JsonServer extends Server<JsonServerBean> {
    private final HttpServer httpServer;

    public JsonServer(String name, String group, boolean validatable, TaskExecutor taskExecutor, DgStrategy dgStrategy, AuthStrategy authStrategy,
            IServiceInitializer initializer, List<Class<?>> ifaces, int workerGroupSize) {
        super(name, group, validatable, taskExecutor, dgStrategy, authStrategy, initializer, ifaces, new JsonServerBean(ifaces));
        @SuppressWarnings("serial")
        List<ChannelHandler> handlerChain = new ArrayList<ChannelHandler>() {
            {
                add(new JsonResponseSerializer());
                add(new HttpServerUrlHandler(new DefaultHandler(HttpResponseStatus.BAD_REQUEST)).register(HttpMethod.POST, JsonConsts.RPC_URI,
                        new JsonRequestDeserializer(JsonServer.this)));
            }
        };
        httpServer = new HttpServer(workerGroupSize, (ch) -> handlerChain.forEach((handler) -> ch.pipeline().addLast(handler)));
    }

    @Override
    public void start(int port) {
        try {
            httpServer.start(port);
        } catch (Exception t) {
            throw new RpcException("Fail to Start server!", t);
        }
    }
}
```

通过FIFRTaskExecutor实现异步，类似于Vine1.x里面的异步[AsyncHandle]({% post_url 2016-06-13-Async_queue_program_model %})（这是在Vine实现的RPC）。其实就是利用netty的ServerBootstrap封了一个HttpServer，JsonRequestDeserializer和JsonResponseSerializer作为ChannelHandler。对于这种封了Server再继承实现DefaultServer的作法便于扩展，如果结合了配置中心的话就可以把bossGroup&workerGroup放到配置中心。关于利用netty实现RPC，我打算另起一篇。

rpc通过一种自定义jsonProtocol实现{JsonBasicHeader.class, JsonExceptionSnap.class, JsonProtoHeader, JsonRequest.class, JsonResponse.class}。

每一次调用通过一个JsonTask。

#### JsonTask

```java
public class JsonTask extends Task<JsonServerBean> {
    private final JsonRequest request;
    private final ChannelHandlerContext ctx;
    @Getter
    private final boolean isKeepAlive;
    @Getter
    private final JsonResponse response;

    public JsonTask(Server<JsonServerBean> server, String ifaceName, String methodName, MultiMessageProducer producer, String requestString,
            JsonRequest request, ChannelHandlerContext ctx, boolean isKeepAlive) {
        super(server, ifaceName, methodName, producer, requestString);
        this.request = request;
        this.ctx = ctx;
        this.isKeepAlive = isKeepAlive;
        this.response = new JsonResponse(request.getVer(), request.getReqId());
    }

    @Override
    protected void prepare() {
        InterfaceInfo interfaceInfo = server.getBean().getServiceInfo().getInterfaceInfo(request.getIface());
        MethodInfo methodInfo = interfaceInfo.getMethodInfo(request.getMethod());
        method = methodInfo.getMethod();
        args = deserializeArgs(methodInfo);
        metas = deserializeMetas(interfaceInfo);
    }

    private Map<String, Object> deserializeArgs(MethodInfo methodInfo) {
        try {
            Map<String, Object> args = methodInfo.getDefaultArgs();
            Map<String, String> argJsons = request.getArgs();
            if (argJsons != null) {
                for (Entry<String, String> entry : argJsons.entrySet()) {
                    args.put(entry.getKey(), JacksonHelper.getMapper().readValue(entry.getValue(), methodInfo.getParamInfo(entry.getKey()).getJavaType()));
                }
            }
            return args;
        } catch (IOException e) {
            throw new JacksonException("Fail to Deserialize Args!", e);
        }
    }

    private Map<String, Object> deserializeMetas(InterfaceInfo interfaceInfo) {
        try {
            Map<String, Object> metas = new HashMap<>();
            Map<String, String> metaJsons = request.getMetas();
            if (metaJsons != null) {
                Map<String, JavaType> metaInfos = interfaceInfo.getMetaInfos();
                for (Entry<String, String> entry : metaJsons.entrySet()) {
                    String key = entry.getKey();
                    JavaType javaType = metaInfos.get(key);
                    if (javaType == null) {
                        throw new VerificationException("Unknown Meta(" + entry.getKey() + ") Found!");
                    }
                    metas.put(key, JacksonHelper.getMapper().readValue(entry.getValue(), javaType));
                }
            }
            return metas;
        } catch (IOException e) {
            throw new JacksonException("Fail to Deserialize Metas!", e);
        }
    }

    @Override
    protected void handleResult(Object result) {
        response.setOriginResult(result);
        ctx.writeAndFlush(this);
    }

    @Override
    protected void handleThrowable(Throwable t) {
        response.setOriginEx(t);
        ctx.writeAndFlush(this);
    }

    @Override
    protected String getClientIpAddress() {
        SocketAddress socketAddress = ctx.channel().remoteAddress();
        if (socketAddress instanceof InetSocketAddress) {
            return ((InetSocketAddress) socketAddress).getAddress().getHostAddress();
        } else {
            return null;
        }
    }
}
```

可以比对[AsyncInvoke接口设计]({% post_url 2016-06-13-Async_queue_program_model %})，预处理、处理、返回结果、异常处理、exit。

未完。。。

