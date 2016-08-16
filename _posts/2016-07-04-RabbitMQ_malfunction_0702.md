---
layout: post
title: 0702订单事故——RabbitMQ并发拥塞
categories: [blog]
tags: [RabbitMQ]
description: 
---

0702订单事故，11:06～11：53补偿单无法配送——RabbitMQ并发拥塞

![监控情况](/img/post/2016-07-04/monitor.jpg)

抖动原因：线程组发送消息时陷入死循环，rabbitmqClient的bug。

ChannelN.java

```
rabbitmqClientVersion＝"3.5.1"
```

```
    protected void close(int closeCode,
                      String closeMessage,
                      boolean initiatedByApplication,
                      Throwable cause,
                      boolean abort)
        throws IOException
    {
        // First, notify all our dependents that we are shutting down.
        // This clears isOpen(), so no further work from the
        // application side will be accepted, and any inbound commands
        // will be discarded (unless they're channel.close-oks).
        Channel.Close reason = new Channel.Close(closeCode, closeMessage, 0, 0);
        ShutdownSignalException signal = new ShutdownSignalException(false,
                                                                     initiatedByApplication,
                                                                     reason,
                                                                     this);
        if (cause != null) {
            signal.initCause(cause);
        }

        BlockingRpcContinuation<AMQCommand> k = new BlockingRpcContinuation<AMQCommand>(){
            @Override
            public AMQCommand transformReply(AMQCommand command) {
                ChannelN.this.finishProcessShutdownSignal();
                return command;
            }};
        boolean notify = false;
        try {
            // Synchronize the block below to avoid race conditions in case
            // connnection wants to send Connection-CloseOK
            synchronized (_channelMutex) {
                startProcessShutdownSignal(signal, !initiatedByApplication, true);
                quiescingRpc(reason, k);
            }

            // Now that we're in quiescing state, channel.close was sent and
            // we wait for the reply. We ignore the result.
            // (It's NOT always close-ok.)
            notify = true;
            k.getReply(-1);//⚠️注意
        } catch (TimeoutException ise) {
            // Will never happen since we wait infinitely
        } catch (ShutdownSignalException sse) {
            if (!abort)
                throw sse;
        } catch (IOException ioe) {
            if (!abort)
                throw ioe;
        } finally {
            if (abort || notify) {
                // Now we know everything's been cleaned up and there should
                // be no more surprises arriving on the wire. Release the
                // channel number, and dissociate this ChannelN instance from
                // our connection so that any further frames inbound on this
                // channel can be caught as the errors they are.
                releaseChannel();
                notifyListeners();
            }
        }
    }
```

多线程在close里跳不出来，导致连异常都没抛就这样hang住了。

pr：![pr](/img/post/2016-07-04/pr.jpg)

升级com.rabbitmq.client，

```
rabbitmqClientVersion＝"3.6.2"
```



```java
    protected void close(int closeCode,
                      String closeMessage,
                      boolean initiatedByApplication,
                      Throwable cause,
                      boolean abort)
        throws IOException, TimeoutException {
        // First, notify all our dependents that we are shutting down.
        // This clears isOpen(), so no further work from the
        // application side will be accepted, and any inbound commands
        // will be discarded (unless they're channel.close-oks).
        Channel.Close reason = new Channel.Close(closeCode, closeMessage, 0, 0);
        ShutdownSignalException signal = new ShutdownSignalException(false,
                                                                     initiatedByApplication,
                                                                     reason,
                                                                     this);
        if (cause != null) {
            signal.initCause(cause);
        }

        BlockingRpcContinuation<AMQCommand> k = new BlockingRpcContinuation<AMQCommand>(){
            @Override
            public AMQCommand transformReply(AMQCommand command) {
                ChannelN.this.finishProcessShutdownSignal();
                return command;
            }};
        boolean notify = false;
        try {
            // Synchronize the block below to avoid race conditions in case
            // connnection wants to send Connection-CloseOK
            synchronized (_channelMutex) {
                startProcessShutdownSignal(signal, !initiatedByApplication, true);
                quiescingRpc(reason, k);
            }

            // Now that we're in quiescing state, channel.close was sent and
            // we wait for the reply. We ignore the result.
            // (It's NOT always close-ok.)
            notify = true;
			      // do not wait indefinitely
            k.getReply(10000);//⚠️注意
        } catch (TimeoutException ise) {
            if (!abort)
                throw ise;
        } catch (ShutdownSignalException sse) {
            if (!abort)
                throw sse;
        } catch (IOException ioe) {
            if (!abort)
                throw ioe;
        } finally {
            if (abort || notify) {
                // Now we know everything's been cleaned up and there should
                // be no more surprises arriving on the wire. Release the
                // channel number, and dissociate this ChannelN instance from
                // our connection so that any further frames inbound on this
                // channel can be caught as the errors they are.
                releaseChannel();
                notifyListeners();
            }
        }
    }
```

