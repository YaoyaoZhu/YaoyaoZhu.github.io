---

layout: post
title: Vine1.x SOA服务的使用
categories: [blog]
tags: [Vine,SOA]
description: Vine Framework
---

# Vine SOA服务的使用

## 1. 使用其他组的服务

**client.json**里配置,将需要调用的第三方的service都注册进来。

```
{
  "clientConfs": [
    {
      "name": "napos.order",
      "threadPoolSize": 500,
      "protocol": "json",
      "timeoutInMillis": 3000,
      "interfaces": [
        "me.ele.napos.miracle.carrier.api.CarrierService",
        "me.ele.napos.miracle.carrier.api.InnerService",
        "me.ele.napos.miracle.carrier.api.OrderPoolService",
        "me.ele.napos.miracle.carrier.api.OrderService",
        "me.ele.napos.miracle.carrier.api.OrderStatsService",
        "me.ele.napos.miracle.carrier.api.PaladinOrderService",
        "me.ele.napos.miracle.carrier.api.PaladinService",
        "me.ele.napos.miracle.carrier.api.RestaurantService"
      ],
      "lbStrategy": "rr",
      "affinity": true,
      "group" : "stable"
    },
    {
      "name": "napos.restaurant",
      "threadPoolSize": 500,
      "protocol": "json",
      "timeoutInMillis": 3000,
      "interfaces": [
        "me.ele.napos.miracle.restaurant.api.CityService",
        "me.ele.napos.miracle.restaurant.api.KeeperService",
        "me.ele.napos.miracle.restaurant.api.KeeperSessionService",
        "me.ele.napos.miracle.restaurant.api.NoticeService",
        "me.ele.napos.miracle.restaurant.api.PhotoService",
        "me.ele.napos.miracle.restaurant.api.RestaurantService",
        "me.ele.napos.miracle.restaurant.api.TopFixedNoticeService"
      ],
      "lbStrategy": "rr",
      "affinity": true,
      "group" : "stable"
    },
    {
        "name": "bpm.bus.policy.soa",
        "protocol": "json",
        "group": "stable",
        "threadPoolSize": 100,
        "timeoutInMillis": 2000,
        "interfaces": [
           "me.ele.bpm.bus.policy.core.service.IPolicyService"
        ],
        "lbStrategy": "rr"
    }
  ]
}
```

上面的服务地址是从huskar里面取得，如果想指定服务地址使用providerList。

```
"providerList":["127.0.0.1:38150"]
```

## 2. 将接口加入Modules

在**ContextListener**的getModules()里添加接口

```
        modules.add(new ESoaClientModule(IPolicyService.class));
        modules.add(new ESoaClientModule("me.ele.napos.miracle.carrier.api"));
		modules.add(new ESoaClientModule("me.ele.napos.miracle.restaurant.api"));
```

上面后两个是批量注册API下的所有接口。

## 3. 本项目的服务Server

**server.json**配置本服务的注册

```
{
  "commonConf": {
    "huskarUrl": "soa-zk.alpha.elenet.me:8020",
    "huskarToken": "eyJhbGciOiJIUzI1NiIsImV4cCI6MjgxNzM5ODA4Njc2LCJpYXQiOjE0NDc4NDA2NzZ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.sRxFS5OivgNh80IiH-bBB7n6-ITMt6QeP1sIRAiV2wc",
    "metricUrl": "statsd.alpha.elenet.me:8125",
    "traceUrl":"etrace-config.alpha.elenet.me:2890"
  },
  "serverConf": {
    "name": "napos.misc",
    "group" : "stable",
    "protocol": "json",
    "port": 38150,
    "threadPoolSize": 100,
    "bufferQueueSize" : 1000,
    "initializer": "me.ele.napos.vine.server.framework.esoa.ESoaServiceInitializer",
    "interfaces": [
      "me.ele.napos.miracle.misc.api.MiscService",
      "me.ele.napos.miracle.misc.api.FeedbackService",
      "me.ele.napos.miracle.misc.api.StatusService",
      "me.ele.napos.miracle.misc.api.TaskService"
    ]
  }
}
```

可以看出使用了注册中心huskar，如果不使用的话"huskarUrl"设为空即可。将提供出去给人访问的接口注册进去，MiscService、FeedbackService等。

## 4. 测试调用

test类

```
public abstract class BaseTest {
    public MiscService miscService;
    public FeedbackService feedbackService;

    @BeforeClass
    public void before() {
        String file = ClassLoader.getSystemResource("client.json").getPath();
        String path = file.replace("/client.json", "");
        System.setProperty("CONF_PATH", path);
        System.out.println("path:" + path);
        Injector injector = Guice.createInjector(
                new ESoaClientModule(MiscService.class),
                new ESoaClientModule(FeedbackService.class)
        );
        miscService = injector.getInstance(MiscService.class);
        feedbackService=injector.getInstance(FeedbackService.class);
    }
}
```



```
public class MiscServiceTest extends BaseTest {
  @Test
    public void getRestaurantBusinessType() {
        List<Integer> restaurantIds = Arrays.asList(17301,11811,808564,731221,1,3,20000);
        restaurantIds.forEach(p -> System.out.println("miscService.getRestaurantBusinessType(i) : " + miscService.getRestaurantBusinessType(p.intValue())));
    }
}
```

测试代码相当于一个client调用，**resources里配置client.json**

```
{
  "commonConf": {
    "huskarUrl": "",
    "huskarToken": "eyJhbGciOiJIUzI1NiIsImV4cCI6MjgxNzM5ODA4Njc2LCJpYXQiOjE0NDc4NDA2NzZ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.sRxFS5OivgNh80IiH-bBB7n6-ITMt6QeP1sIRAiV2wc",
    "metricUrl": "statsd.alpha.elenet.me:8125",
    "traceUrl": "etrace-config.alpha.elenet.me:2890"
  },
  "clientConfs": [{
    "name":"napos.misc",
    "group": "localDev",
    "protocol": "json",
    "timeoutInMillis": 200000000,
    "threadPoolSize": 100,
    "interfaces": [
      "me.ele.napos.miracle.misc.api.MiscService",
      "me.ele.napos.miracle.misc.api.FeedbackService",
      "me.ele.napos.miracle.misc.api.StatusService",
      "me.ele.napos.miracle.misc.api.TaskService"
    ],
    "providerList":["127.0.0.1:38150"]
  }
  ]
}
```

注意地址为本机，端口为38150与server.json配置一致。