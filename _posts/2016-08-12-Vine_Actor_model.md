---
layout: post
title: Vine Actor模型
categories: [blog]
tags: [Java]
description: 
---

Vine Actor模型

和并发的Actor有区别的，那个也会单独写一篇。

### Actor

什么是Actor？资源如redis、db、cache等，Engine如HttpEngine、ConsumerEngine、SoaEngine等,Processor。

```java
public interface Actor {
    default void initialize(ActorConfig config) {
    }
    default void validate() {
    }
}
```

### ActorMetaSettings

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ActorMetaSettings {
    Class<? extends ActorConfig> configType() default ActorConfig.class;

    Class<? extends Annotation> settingsType() default Annotation.class;

    Class<? extends ActorSettingsX> settingsXType() default ActorSettingsX.class;

    Class<? extends Atom>[] atomTypes() default {};
}
```

对于impl了Actor的类通过加上@ActorMetaSettings的注解完成参数配置。

#### ActorConfig

ActorConfig接口的实现类，作为资源配置一般有：

```java
    private String url;
    private Integer timeoutMills;
    private Integer poolMaxTotal;
    private Integer poolMaxIdle;
    private Integer poolMinIdle;
    private Integer poolWaitMills;
```

而注解如RedisSettings、DatabaseSettings（extends Annotation）与ActorSettingsX，都会merge进ActorConfig。

#### Annotation

RedisSettings、DatabaseSettings等

#### ActorSettingsX

DatabaseSettingsX、RedisSettingsX等，从Huskar读，将会覆盖merge到config。

#### Atom

```java
public interface Atom {
    default void recycle() {}
}
```

对于DB：SessionHolder

对于Keeper：SessionKeeper



#### Vine.class

```java
    private static final Context context = ContextImpl.initialize();	
	public static ActorProvider getActorProvider() {
        return context.getActorProvider();
    }

    public static <A extends Actor> A getActor(Class<A> actorType) {
        return context.getActorProvider().getActor(actorType);
    }

    public static <A extends Actor> A getWrappedActor(Class<A> wrappedActorType) {
        return context.getActorProvider().getWrappedActor(wrappedActorType);
    }
```

```java
interface Context {
    EnvRegistry getEnvRegistry();
    ConfigManager getConfigManager();
    AtomProvider getAtomProvider();
    ActorProvider getActorProvider();
    LoggerProvider getLoggerProvider();
    MeterProvider getMeterProvider();
}
```

也就是说，当我们需要使用Actor的时候，只要Vine.getWrappedActor(clazz)即可获得该Actor实例，如Vine.getWrappedActor(Redis.class)，即可获得实例，而实例化的配置通过注解实现。