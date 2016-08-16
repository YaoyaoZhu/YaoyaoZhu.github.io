---
layout: post
title: Vine_legacy SOA 解析
categories: [blog]
tags: [Vine,SOA]
description: 
---



## mainClass：Container

```java
 public static void main(String[] args) {
        try {
            CoreBootstrap.getInstance().include((args != null && args.length > 0) ? args : new String[] { CoreConsts.DEFAULT_CONFIG_PATH }).start();
        } catch (Throwable t) {
            logger.error("Throwable Occurs in Container!", t);
            CoreBootstrap.getInstance().stop(-1);
        }
    }
```

读取了jvm启动参数，有如下路径配置（build.gradle）：

```java
applicationDefaultJvmArgs=["-Dport=8907",
                           "-DCONF_PATH=$rootDir/config",
                           "-DAPPID=napos.miracle.version",
                           "-Delog.config=$rootDir/config/elog.xml",
                           "-DTEAM=napos",
                           "-Dlistener=me.ele.napos.miracle.version.server.MiracleVersionContextListener"]
```

## 进入soa的CoreBootstrap



```java
public class CoreBootstrap {
    @Getter
    private final static CoreBootstrap instance = new CoreBootstrap();
    private final SOAConf myConf = new SOAConf();
    public CoreBootstrap include(String... paths) {
        if (paths != null) {
            for (String path : paths) {
                include(CoreInitializer.readConf(path));//read config path
            }
        }
        return this;
    }
    public void start() {
        CoreInitializer.init(myConf);
    }
    //omit superfluous code
}
```

## 进入soa的CoreInitializer

由CoreBootstrap.include()进入CoreInitializer.readConf()读取配置文件，logger的日志在启动的时候打出来。

```java
    public static SOAConf readConf(String path) {
        try {
            logger.info("Start to Read Configuration File({})!", path);
            SOAConf soaConf = JacksonHelper.getMapper().readValue(new File(path), SOAConfBean.class).convert();
            logger.info("Succeed to Read Configuration File!");
            return soaConf;
        } catch (IOException e) {
            throw new CoreException(e);
        }
    }
```

由CoreBootstrap.start()进入CoreInitializer.init()进行服务端或者客户端的初始化。

```java
    public static void init(SOAConf soaConf) {
        execute(() -> {
            init(soaConf.getCommonConf());
            init(soaConf.getClientConfs());
            init(soaConf.getServerConf());
        });
    } 
```

init(soaConf.getCommonConf())读取Huskar配置和将metricUrl、traceUrl初始化

init(soaConf.getClientConfs())soa客户端配置，init(soaConf.getServerConf())服务端配置。

**ps：这里的execute(() -> {//todo})使用了函数式接口**

```java
    @FunctionalInterface
    private static interface Initialization {
        void init();
    }

    private static void execute(Initialization initialization) {
        try {
            initialization.init();
        } catch (Throwable t) {
            logger.error("Throwable Occurs during initialization!", t);
            exit(-1);
        }
    }
```

服务端启动

```java
    public static void init(ServerConf serverConf) {
        execute(() -> {
            if (serverConf != null) {
                logger.info("Start to Initialize Server!");
                validate(serverConf);
                if (!commonInitialized) {
                    throw new CoreException("Server should be initialized after Common!");
                } else if (serverInitialized) {
                    throw new CoreException("Server has already been initialized!");
                } else {
                    serverInitialized = true;

                    if (serverConf.getExitDelayInMillis() > 0) {
                        exitDelayInMillis = serverConf.getExitDelayInMillis();
                    }

                    AgentConfiguration.setServiceName(serverConf.getName());
                    try {
                        HuskarHandle.get().initMy(serverConf.getName(), serverConf.getGroup());
                    } catch (HuskarException e) {
                        throw new CoreException(e);
                    }

                    serverConf.getInitializer().init(); //
                    startBizServer(serverConf);
                    startDaemonServer(serverConf);
                    registerService(serverConf);
                    new Thread(() -> new AcvClient(serverConf.getInterfaces()).post()).start();

                    logger.info("Succeed to Initialize Server!");
                }
            }
        });
    }
```

serverConf.getInitializer().init()的接口实现是ESoaServiceInitializer.init() ，里面加载路径配置的ContextListener。

