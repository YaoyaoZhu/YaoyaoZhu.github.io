---
layout: post
title: Async异步队列编程模型
categories: [blog ]
tags: [Java]
description: 应对大并发量的缓冲
---

# Async异步队列编程模型

## 应用场景

**Before：**ContextRequest——>workflow(contextRequest)

​	contextRequest直接随着本线程进入处理流程

**After:** ContextRequest——>TaskImpl(contextRequest, asyncInvokeImpl)——>AsyncHandle.offer(task)………Worker——>AsyncHandle.poll(task)——>executor.execute(task)

​	contextRequest和封装处理它的流程的实例作为参数进入TaskImpl的构造器以获得task实例，再将task放入AsyncHandle里的阻塞队列里。AsyncHandle在实例化时启动独立线程不断将task取出来给Executor的线程组处理。

**需求**：RPC中，将Acceptor(bossGroup)与workGroup异步并以队列存放task，以提高性能。
**压测结果** qps>20k/s



## AsyncHandle设计

```java
public class AsyncHandle {

    private static final int DEFAULT_QUEUE_MAX_SIZE = 500;
    private static final int DEFAULT_WORKER_MIN_SIZE = Runtime.getRuntime().availableProcessors();
    private static final int DEFAULT_WORKER_MAX_SIZE = Runtime.getRuntime().availableProcessors() * 2;

    private int workerCount = DEFAULT_WORKER_MAX_SIZE;

    private BlockingQueue<AsyncTask> tasks = null;

    private Executor executor = new ThreadPoolExecutor(
            DEFAULT_WORKER_MIN_SIZE,
            workerCount,
            60L, TimeUnit.SECONDS,
            new SynchronousQueue<>(),
            new VThreadFactory("async-invoker"),
            new ThreadPoolExecutor.CallerRunsPolicy());

    /**
     *  默认队列大小
     * @param workerCount 处理任务线程数
     */
    public AsyncHandle(int workerCount) {
        this(DEFAULT_QUEUE_MAX_SIZE, workerCount);
    }

    /**
     * 默认参数 queueSize 500; workerCount CUP * 2
     */
    public AsyncHandle() {
        this(DEFAULT_QUEUE_MAX_SIZE, DEFAULT_WORKER_MAX_SIZE);
    }

    /**
     * 默认 FIFRBlockingQueue 作为队列容器
     * @param queueMaxSize 队列最大容量
     * @param workerCount 处理队列的线程数
     */
    public AsyncHandle(int queueMaxSize, int workerCount) {
        this.workerCount = workerCount;
        tasks = new FIFRBlockingQueue<>(queueMaxSize, new VOfferPolicy());
        new Thread(new AsyncWorker()).start();
    }

    /**
     * 可指定队列作为容器
     * @param workerCount 处理线程数
     * @param queue 队列
     */
    public AsyncHandle(int workerCount, BlockingQueue<AsyncTask> queue) {
        this.workerCount = workerCount;
        this.tasks = queue;
        new Thread(new AsyncWorker()).start();
    }

    /**
     * Task 入队
     * @param task 任务
     * @throws AsyncInvokerException
     */
    public void offer(AsyncTask task) throws AsyncInvokerException {
        tasks.offer(task);
    }

    /**
     * 处理任务类
     */
    class AsyncWorker implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    AsyncTask task = tasks.poll(100, TimeUnit.MILLISECONDS);
                    if(task == null) {
                        continue;
                    }
                    executor.execute(task);
                } catch (Exception e) {
                }
            }
        }
    }
}
```

FIFRBlockingQueue作为容器，是根据jdk提供的FIFOBlockingQueue加入移除策略改写的，为了满足日后业务需求提供了指定容器的构造器。



## AsyncTask设计

```java
import lombok.Data;

@Data
public class AsyncTask<T> implements Runnable{

    private AsyncInvoke<T> invoke;
    private long currentTime;
    private T ctx;

    public void init(AsyncInvoke<T> invoke, long currentTime, T t) {
        this.invoke = invoke;
        this.currentTime = currentTime;
        this.ctx = t;
    }

    public void cancelTask() {
        try {
            if (invoke.enter(ctx)) {
                invoke.exceptionCatch(ctx, new AsyncInvokerException("Task was abandon"));
            }
        }catch (Exception e) {
            invoke.exceptionCatch(ctx, e);
        }finally {
            invoke.exit(ctx);
        }
    }

    @Override
    public void run() {
        try {
            if(invoke.enter(ctx)) {
                ctx = invoke.run(ctx);
                invoke.messageReceiver(ctx);
            }
        }catch (Throwable t) {
            invoke.exceptionCatch(ctx, t);
        }finally {
            invoke.exit(ctx);
        }
    }
}
```

init()用于封装task，cancelTask()在移除策略里使用，run()继承Runnable表明这是可以提供给Worker执行的线程组的任务



## AsyncInvoke接口设计
### Interface

```java
public interface AsyncInvoke<T> {
    default boolean enter(T t) throws Exception {
        return true;
    }

    T run(T t) throws Exception;

    default T messageReceiver(T t) throws Exception {
        return t;
    }

    void exceptionCatch(T t, Throwable e);

    default void exit(T t) {

    }
}
```

enter()作为预处理方法，与主要业务处理方法run()分开，因为即使任务因为超时等原因取消了也需要走流程返回ncp的response。exceptionCatch()是异常处理流程，exit()是放在finally里。

### impl

```
public class VineAsyncInvokeImpl implements AsyncInvoke<CtxRequest> {
     @Inject
     private Injector injector;
     @Inject
     private RequestScope scope;
     private boolean ctxClose=false;
     @Inject
     public VineAsyncInvokeImpl(){}
 
     private Log logger = VineLogFactory.getLog(VineAsyncInvokeImpl.class);
     FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK);
     HttpAdapter adapter = (HttpAdapter) injector.getInstance(VineGlobal.getGlobal().getAdapterClass());
 
     @Override
     public boolean enter(CtxRequest ctxRequest) throws ServiceException {
         scope.enter();
         InetSocketAddress insocket = (InetSocketAddress) ctxRequest.getChannelHandlerContext().channel()
                 .remoteAddress();
         adapter.initialize(ctxRequest.getRequest(), response, insocket.getAddress().getHostAddress());
         return adapter.preProcess();//
     }
 
     @Override
     public CtxRequest run(CtxRequest ctxRequest) throws ServiceException {
         VineFilterChain filterChain = injector.getInstance(VineFilterChain.class);
         filterChain.execute();
 
 
         return ctxRequest;
     }
 
     @Override
     public CtxRequest messageReceiver(CtxRequest ctxRequest) throws ServiceException {
         return ctxRequest;
     }
 
     @Override
     public void exceptionCatch(CtxRequest ctxRequest, Throwable e) {
         ctxClose=true;
         if (e instanceof InvalidInvocationException) {
             adapter.getResponse().setError((InvalidInvocationException) e);
         } else {
             adapter.getResponse().setError((new ServerErrorException()));
         }
 
     }
 
     @Override
     public void exit(CtxRequest ctxRequest) {
         try {
             try {
                 adapter.postProcess();
             } catch (InvalidMethodException e) {
                 response.setStatus(HttpResponseStatus.BAD_REQUEST);
                 logger.error("Invalid Method Exception ", e);
             } catch (InvalidNotFindPathException e) {
                 response.setStatus(HttpResponseStatus.NOT_FOUND);
                 logger.error("Invalid Not Find Path Exception", e);
             } catch (Exception e) {
                 response.setStatus(HttpResponseStatus.INTERNAL_SERVER_ERROR);
                 logger.error("adapter post process error", e);
             }
             boolean isKeepAlive = HttpHeaders.isKeepAlive(ctxRequest.getRequest());
             if (isKeepAlive) {
                 response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
             }
             ChannelFuture future = ctxRequest.getChannelHandlerContext().writeAndFlush(response);
 
             if (!isKeepAlive) {
                 try {
                     future.sync();
                 } catch (InterruptedException e) {
                     e.printStackTrace();
                 }
                 ctxRequest.getChannelHandlerContext().close();
             }
             if(ctxClose){
                 ctxRequest.getChannelHandlerContext().close();
             }
         } finally {
             scope.exit();
         }
 
     }
 }
```

[^ncp]: vine的自定义rpc通讯协议



## 杂项

### FIFRBlockingQueue简要

```java
public class FIFRBlockingQueue<T> extends AbstractQueue<T> implements BlockingQueue<T> {
    final OfferPolicy<T> policy;
    public FIFRBlockingQueue(int capacity, OfferPolicy<T> policy) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(false);
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
        this.policy = policy;
    }

    public boolean offer(T obj) {
        checkNotNull(obj);
        T lastNode = null;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length) {
                lastNode = rpqueue(obj);
            } else {
                enqueue(obj);
            }
        } finally {
            lock.unlock();
        }

        if (lastNode != null) {
            policy.cancel(lastNode);
        }
        return true;
    }

    public T poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
}
```

### OfferPolicy接口

```java
public interface OfferPolicy<T> {
    public static final OfferPolicy<Object> Default = new OfferPolicy<Object>() {
        @Override
        public void cancel(Object t) {}
    };
    void cancel(T t);
}
```

### VofferPolicy实现

```java
public class VOfferPolicy implements OfferPolicy<AsyncTask> {
    @Override
    public void cancel(AsyncTask asyncTask) {
        asyncTask.cancelTask();
    }
}
```

### VThreadFactory

```java
public class VThreadFactory implements ThreadFactory {

    private final String threadNamePrefix;
    private final AtomicInteger number = new AtomicInteger(0);

    public VThreadFactory(String threadNamePrefix) {
        this.threadNamePrefix = threadNamePrefix;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setName(threadNamePrefix + "-" + number.getAndIncrement());
        return thread;
    }
}
```

