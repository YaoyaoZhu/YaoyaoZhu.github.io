---
layout: post
title: 问题定位——Exception总结
categories: [blog]
tags: [Java]
description: 
---
# Exception

异常处理：

错误：

```java
    public MarketManager getMarketManager(int shopId) throws ServiceException {
        PackageInfoDto packageInfoDto = getManagerByRestaurant(shopId);
        MarketManager marketManager = new MarketManager();
        marketManager.setName(packageInfoDto.getPackageManagerName());
        marketManager.setMail(packageInfoDto.getPackageManagerEmail());
        marketManager.setMobile(String.valueOf(packageInfoDto.getPackageManagerPhone()));
        return marketManager;
    }

    private PackageInfoDto getManagerByRestaurant(int shopId) {
        try {
            return policyService.getPackageInfoByRstId(shopId);
        } catch (ServiceException e) {
            logger.info("getManagerByRestaurant service exception", e);
            return null;
        } catch (ServerException e) {
            logger.error("getManagerByRestaurant error", e);
            return null;
        }
    }
```

正确：

```java
    public MarketManager getMarketManager(int shopId) {
        PackageInfoDto packageInfoDto = getManagerByRestaurant(shopId);
        if (packageInfoDto == null) return null;
        MarketManager manager = new MarketManager();
        if (packageInfoDto.getPackageManagerPhone() > 0) {
            manager.setName(packageInfoDto.getPackageManagerName());
            manager.setMobile(String.valueOf(packageInfoDto.getPackageManagerPhone()));
            manager.setMail(packageInfoDto.getPackageManagerEmail());
        } else {
            manager.setName(packageInfoDto.getCampManagerName());
            manager.setMobile(String.valueOf(packageInfoDto.getCampManagerPhone()));
            manager.setMail(packageInfoDto.getCampManagerEmail());
        }
        return manager;
    }

    private PackageInfoDto getManagerByRestaurant(int shopId) {
        try {
            return policyService.getPackageInfoByRstId(shopId);
        } catch (me.ele.contract.exception.ServiceException e) {
            logger.info("getManagerByRestaurant service exception", e);
            return null;
        } catch (ServerException e) {
            logger.error("getManagerByRestaurant error", e);
            return null;
        }
    }
```

空指针这样没必要抛的异常就直接返回null，将异常堆栈信息打印出来即可，不要让上层再catch。



## ServiceException,RuntimeException,Exception

ServiceException定义为正常业务的异常，当服务有依赖，那么ServiceException捕获到了可以根据业务来决定如何处理。具体地说，如果要挂一起挂，那就不让它过，new一个ServiceException；如果业务没影响，打个日志就可以了。

RuntimeException属于非异常，是不可预知的。需要处理，可以根据业务耦合度判断是否new一个exception，还是完全不用管。

Exception是root，所以在不同系统服务调用的时候，有一些异常没有代码提示catch，所以要使用Exception来捕，特别的，这也是替代UDP模式的一种形式。



## Debug Exception总结

构造初始化的时候就失败了，可能是Guice注入实例时失败了。

```
at com.google.common.reflect.Invokable$ConstructorInvokable.invokeInternal(Invokable.java:242)
	at com.google.common.reflect.Invokable.invoke(Invokable.java:102)
	at me.ele.napos.vine.common.reflect.VInvoke.newInstance(VInvoke.java:51)
```

