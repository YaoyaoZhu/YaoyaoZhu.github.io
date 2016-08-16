---
layout: post
title: RabbitMQ生产者模型
categories: [blog]
tags: [RabbitMQ, Java]
description: 
---
# RabbitMQ生产者模型

## RabbitMQ连接池设计

采用阻塞队列作为连接池的存储结构，避免使用代理，再生产者那一层完成归还连接的操作，提高性能。连接池connection总数固定，使用委托的自动恢复连接的机制。

**初始化**

```java
public RabbitConnectionPool(String uri, int rabbitPoolSize, long waitTimes) {
        try {
            rtnConnTask();
            ConnectionFactory factory = new ConnectionFactory();
            factory.setUri(uri);
            factory.setAutomaticRecoveryEnabled(true);
            factory.setRequestedHeartbeat(5);
            factory.setConnectionTimeout(2000);
            int poolSize = rabbitPoolSize == 0 ? DEFAULT_POOL_SIZE : rabbitPoolSize;
            waitTime = waitTimes == 0 ? DEFAULT_WAIT_TIME : waitTimes;
            for (int i = 0; i < poolSize; i++) {
                connList.add(factory.newConnection());
            }
        } catch (Exception e) {
            logger.error("create rabbitmq pool fail.", e);
            throw new RuntimeException(e);
        }
    }
```



通过两个队列分别存放正常的和异常的connection

```java
   private LinkedBlockingDeque<Connection> connList = new LinkedBlockingDeque<Connection>();
    private LinkedBlockingDeque<Connection> connClosedList = new LinkedBlockingDeque<Connection>();
```

**获得connection的实例**

通过pool的getConnection()

```java
    public Connection getConnection() throws VineRMQException {
        Connection connection = null;
        try {
            connection = connList.poll(waitTime, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            throw new VineRMQException(e);
        }
        if (connection == null) {
            logger.error("rabbitMQ connections pool full and conn are all in using");
            throw new VineRMQException("rabbitMQ connections pool full and conn are all in using");
        }
        if (!connection.isOpen()) {
            connClosedList.offer(connection);
            return getConnection();
        }
        return connection;
    }
```

**将恢复的异常connection放入正常的queue的task**

```java
    public void rtnConnTask() {
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                try {
                    rtnConnToClosedConnList();
                } catch (VineRMQException e) {
                    logger.error("rtnConnToClosedConnList failed", e);
                    throw new RuntimeException(e);
                }
            }
        }, DEFAULT_RETURN_INTERVAL_MILLS, DEFAULT_RETURN_INTERVAL_MILLS);
    }
```

```java
    private void rtnConnToClosedConnList() throws VineRMQException {
        Connection connection = connClosedList.poll();
        if (connection == null) {
            return;
        } else {
            boolean flag = connection.isOpen() ? connList.offer(connection) : connClosedList.offer(connection);
            if (!flag) {
                logger.error("RabbitMQConnectionLost : rtnConnToClosedConnList cause ERROR");
            }
        }

    }
```

**归还connection的方法**

```java
    public void returnConnection(Connection connection) {
        if (!connList.offer(connection)) logger.error("RabbitMQConnectionLost : returnConnection cause ERROR");
    }
```



## 生产者设计

**初始化**

```java
    public VineMQProducer() {
        if (!inited) {
            try {
                initialized(false);
            } catch (Exception e) {
                logger.error("MQ Producer init failed, configKey:" + getConfigKey(), e);
            }
        }
    }

    public void initialized(boolean throwExc) {
        try {
            Config conf = DefaultConfig.getConfig();
            config = conf.get(getConfigKey(), MQProducerConfig.class);
            connPool = new RabbitConnectionPool(config.getUri(), config.getPoolSize(), config.getWaitTime());
            inited = true;
        } catch (Exception e) {
            if (throwExc) {
                throw e;
            }
            logger.warn("MQ Producer init failed, configKey:" + getConfigKey(), e);
            inited = true;
        }
    }
```

**发送消息**

消息的发送可以传入routingKey进行路由

```java
    public void send(String message, String routingKey) throws VineRMQException {
        Channel channel = null;
        try {
            Connection conn = connPool.getConnection();

            try {
                channel = conn.createChannel();
                if (channel == null) {
                    throw new VineRMQException("connection is null");
                }
            } finally {
                connPool.returnConnection(conn);
            }

            if (config.getExchange() != null) {
                if (routingKey == null) {
                    channel.basicPublish(config.getExchange(), config.getRoutingKey(), null, message.getBytes());
                } else {
                    channel.basicPublish(config.getExchange(), routingKey, null, message.getBytes());
                }
            }

        } catch (Exception e) {
            logger.error("send message fail. message:" + message);
            throw new VineRMQException(e);
        } finally {
            if (channel != null && channel.isOpen()) {
                try {
                    channel.close();
                } catch (IOException e) {
                    throw new VineRMQException(e);
                }
            }
        }
    }
```

**线程组发送**

每个线程都单独获得一个channel，避免了排队，虽让channel本身是线程安全的

```java
    private ExecutorService executorService = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>(10_000));
```

```java
    public void asyncSend(String message, String routingKey) {
        executorService.execute(() -> {
            try {
                send(message, routingKey);
            } catch (VineRMQException e) {
            logger.error(message + "send failed, exception: " + e);
            }
        });
    }
```



## 踩过的坑

**坑1:** 一开始是设计了，connection和channel的双连接池的设计，但是实际测试发现channel处理能力非常强大，2000q/s也只需要一个connection产生平均1.2个channel就足够使用了，所以舍弃了该方案。

**坑2:**连接池使用了代理的方式，在压测的时候没发现问题，上了生产，结果订单发生了错发！错发！非常可怕的bug，我仔细看了代码，没发现问题，在并发量小时候是没有问题的，结果量大就出现这种问题了，推测可能是RabbitMQ的问题。

## 心得

越是简单的设计越是好设计。

