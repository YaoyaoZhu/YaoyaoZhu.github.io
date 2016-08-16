---
layout: post
title: RabbitMQ Analysis
categories: [blog]
tags: [RabbitMQ]
description: 
---
# RabbitMQ Analysis

## AMQP model

**AMQP messaging 中的基本概念**

[![RabbitMQ与AMQP协议详解](/img/2016-06-13/amqp_protcol.png) 

Broker:    接收和分发消息的应用，RabbitMQ Server就是Message Broker。

Virtual host:    出于多租户和安全因素设计的，把AMQP的基本组件划分到一个虚拟的分组中，类似于网络中的namespace概念。当多个不同的用户使用同一个RabbitMQ server提供的服务时，可以划分出多个vhost，每个用户在自己的vhost创建exchange／queue等。

Connection: publisher／consumer和broker之间的TCP连接。断开连接的操作只会在client端进行，Broker不会断开连接，除非出现网络故障或broker服务出现问题。

Channel: 如果每一次访问RabbitMQ都建立一个Connection，在消息量大的时候建立TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id帮助客户端和message broker识别channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP connection的开销。

Exchange: message到达broker的第一站，根据分发规则，匹配查询表中的routing key，分发消息到queue中去。常用的类型有：direct (point-to-point), topic (publish-subscribe) and fanout (multicast)。

Queue:    消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。

Binding: exchange和queue之间的虚拟连接，binding中可以包含routing key。Binding信息被保存到exchange中的查询表中，用于message的分发依据。

本段转载自(http://www.openstack.cn/?p=4702)



## AMQP的协议栈

AMQP协议本身包括三层：

![img](https://dn-sdkcnssl.qbox.me/editor/kqQkd1DTXwpK9_9mbBY3.png)

1.Module Layer，位于协议最高层，主要定义了一些供客户端调用的命令，客户端可以利用这些命令实现自己的业务逻辑，例如，客户端可以通过queue.declare声明一个队列，利用consume命令获取一个队列中的消息。

2.Session Layer，主要负责将客户端的命令发送给服务器，在将服务器端的应答返回给客户端，主要为客户端与服务器之间通信提供可靠性、同步机制和错误处理。

3.Transport Layer，主要传输二进制数据流，提供帧的处理、信道复用、错误检测和数据表示。



## RabbitMQ使用场景

学习RabbitMQ的使用场景，来自官方教程：[https://www.rabbitmq.com/getstarted.html](https://www.rabbitmq.com/getstarted.html)

### 场景1：单发送单接收

使用场景：简单的发送与接收，没有特别的处理。

![img](https://dn-sdkcnssl.qbox.me/editor/dF4cPtIzNVjxpCFfPgAA.png)

Producer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; public class Send {        private final static String QUEUE_NAME = "hello";   public static void main(String[] argv) throws Exception {                      ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.queueDeclare(QUEUE_NAME, false, false, false, null);     String message = "Hello World!";     channel.basicPublish("", QUEUE_NAME, null, message.getBytes());     System.out.println(" [x] Sent '" + message + "'");          channel.close();     connection.close();   } }
```

Consumer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.QueueingConsumer; public class Recv {          private final static String QUEUE_NAME = "hello";     public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.queueDeclare(QUEUE_NAME, false, false, false, null);     System.out.println(" [*] Waiting for messages. To exit press CTRL+C");          QueueingConsumer consumer = new QueueingConsumer(channel);     channel.basicConsume(QUEUE_NAME, true, consumer);          while (true) {       QueueingConsumer.Delivery delivery = consumer.nextDelivery();       String message = new String(delivery.getBody());       System.out.println(" [x] Received '" + message + "'");     }   } }
```

### 场景2：单发送多接收

使用场景：一个发送端，多个接收端，如分布式的任务派发。为了保证消息发送的可靠性，不丢失消息，使消息持久化了。同时为了防止接收端在处理消息时down掉，只有在消息处理完成后才发送ack消息。

![img](https://dn-sdkcnssl.qbox.me/editor/Yqm1dGYMJ-wP6ySv62Bi.png)

Producer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.MessageProperties; public class NewTask {      private static final String TASK_QUEUE_NAME = "task_queue";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();          channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);          String message = getMessage(argv);          channel.basicPublish( "", TASK_QUEUE_NAME,                  MessageProperties.PERSISTENT_TEXT_PLAIN,                 message.getBytes());     System.out.println(" [x] Sent '" + message + "'");          channel.close();     connection.close();   }        private static String getMessage(String[] strings){     if (strings.length < 1)       return "Hello World!";     return joinStrings(strings, " ");   }        private static String joinStrings(String[] strings, String delimiter) {     int length = strings.length;     if (length == 0) return "";     StringBuilder words = new StringBuilder(strings[0]);     for (int i = 1; i < length; i++) {       words.append(delimiter).append(strings[i]);     }     return words.toString();   } }
```

发送端和场景1不同点：

1、使用“task_queue”声明了另一个Queue，因为RabbitMQ不容许声明2个相同名称、配置不同的Queue

2、使"task_queue"的Queue的durable的属性为true，即使消息队列durable

3、使用MessageProperties.PERSISTENT_TEXT_PLAIN使消息durable

When RabbitMQ quits or crashes it will forget the queues and messages unless you tell it not to. Two things are required to make sure that messages aren't lost: we need to mark both the queue and messages as durable.

Consumer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.QueueingConsumer;    public class Worker {   private static final String TASK_QUEUE_NAME = "task_queue";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();          channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);     System.out.println(" [*] Waiting for messages. To exit press CTRL+C");          channel.basicQos(1);          QueueingConsumer consumer = new QueueingConsumer(channel);     channel.basicConsume(TASK_QUEUE_NAME, false, consumer);          while (true) {       QueueingConsumer.Delivery delivery = consumer.nextDelivery();       String message = new String(delivery.getBody());              System.out.println(" [x] Received '" + message + "'");       doWork(message);       System.out.println(" [x] Done");       channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);     }            }      private static void doWork(String task) throws InterruptedException {     for (char ch: task.toCharArray()) {       if (ch == '.') Thread.sleep(1000);     }   } }
```

接收端和场景1不同点：

1、使用“task_queue”声明消息队列，并使消息队列durable

2、在使用channel.basicConsume接收消息时使autoAck为false，即不自动会发ack，由channel.basicAck()在消息处理完成后发送消息。

3、使用了channel.basicQos(1)保证在接收端一个消息没有处理完时不会接收另一个消息，即接收端发送了ack后才会接收下一个消息。在这种情况下发送端会尝试把消息发送给下一个not busy的接收端。

注意点：

1）It's a common mistake to miss the basicAck. It's an easy error, but the consequences are serious. Messages will be redelivered when your client quits (which may look like random redelivery), but RabbitMQ will eat more and more memory as it won't be able to release any unacked messages.

2）Note on message persistence

Marking messages as persistent doesn't fully guarantee that a message won't be lost. Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync(2) for every message -- it may be just saved to cache and not really written to the disk. The persistence guarantees aren't strong, but it's more than enough for our simple task queue. If you need a stronger guarantee you can wrap the publishing code in atransaction.

3）Note about queue size

If all the workers are busy, your queue can fill up. You will want to keep an eye on that, and maybe add more workers, or have some other strategy.

### 场景3：Publish/Subscribe

使用场景：发布、订阅模式，发送端发送广播消息，多个接收端接收。

![img](https://dn-sdkcnssl.qbox.me/editor/IsTimJHOKpuX0HGkh0nY.png)

Producer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; public class EmitLog {   private static final String EXCHANGE_NAME = "logs";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.exchangeDeclare(EXCHANGE_NAME, "fanout");     String message = getMessage(argv);     channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());     System.out.println(" [x] Sent '" + message + "'");     channel.close();     connection.close();   }      private static String getMessage(String[] strings){     if (strings.length < 1)             return "info: Hello World!";     return joinStrings(strings, " ");   }      private static String joinStrings(String[] strings, String delimiter) {     int length = strings.length;     if (length == 0) return "";     StringBuilder words = new StringBuilder(strings[0]);     for (int i = 1; i < length; i++) {         words.append(delimiter).append(strings[i]);     }     return words.toString();   } }
```

发送端：

发送消息到一个名为“logs”的exchange上，使用“fanout”方式发送，即广播消息，不需要使用queue，发送端不需要关心谁接收。

Consumer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.QueueingConsumer; public class ReceiveLogs {   private static final String EXCHANGE_NAME = "logs";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.exchangeDeclare(EXCHANGE_NAME, "fanout");     String queueName = channel.queueDeclare().getQueue();     channel.queueBind(queueName, EXCHANGE_NAME, "");          System.out.println(" [*] Waiting for messages. To exit press CTRL+C");     QueueingConsumer consumer = new QueueingConsumer(channel);     channel.basicConsume(queueName, true, consumer);     while (true) {       QueueingConsumer.Delivery delivery = consumer.nextDelivery();       String message = new String(delivery.getBody());       System.out.println(" [x] Received '" + message + "'");        }   } }
```

接收端：

1、声明名为“logs”的exchange的，方式为"fanout"，和发送端一样。

2、channel.queueDeclare().getQueue();该语句得到一个随机名称的Queue，该queue的类型为non-durable、exclusive、auto-delete的，将该queue绑定到上面的exchange上接收消息。

3、注意binding queue的时候，channel.queueBind()的第三个参数Routing key为空，即所有的消息都接收。如果这个值不为空，在exchange type为“fanout”方式下该值被忽略！

### 场景4：Routing (按路线发送接收)

使用场景：发送端按routing key发送消息，不同的接收端按不同的routing key接收消息。

![img](https://dn-sdkcnssl.qbox.me/editor/5HSpU2l8E7RJoyw-ZaLM.png)

Producer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; public class EmitLogDirect {   private static final String EXCHANGE_NAME = "direct_logs";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.exchangeDeclare(EXCHANGE_NAME, "direct");     String severity = getSeverity(argv);     String message = getMessage(argv);     channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());     System.out.println(" [x] Sent '" + severity + "':'" + message + "'");     channel.close();     connection.close();   }      private static String getSeverity(String[] strings){     if (strings.length < 1)             return "info";     return strings[0];   }   private static String getMessage(String[] strings){      if (strings.length < 2)             return "Hello World!";     return joinStrings(strings, " ", 1);   }      private static String joinStrings(String[] strings, String delimiter, int startIndex) {     int length = strings.length;     if (length == 0 ) return "";     if (length < startIndex ) return "";     StringBuilder words = new StringBuilder(strings[startIndex]);     for (int i = startIndex + 1; i < length; i++) {         words.append(delimiter).append(strings[i]);     }     return words.toString();   } }
```

发送端和场景3的区别：

1、exchange的type为direct

2、发送消息的时候加入了routing key

Consumer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.QueueingConsumer; public class ReceiveLogsDirect {   private static final String EXCHANGE_NAME = "direct_logs";   public static void main(String[] argv) throws Exception {     ConnectionFactory factory = new ConnectionFactory();     factory.setHost("localhost");     Connection connection = factory.newConnection();     Channel channel = connection.createChannel();     channel.exchangeDeclare(EXCHANGE_NAME, "direct");     String queueName = channel.queueDeclare().getQueue();          if (argv.length < 1){       System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");       System.exit(1);     }          for(String severity : argv){           channel.queueBind(queueName, EXCHANGE_NAME, severity);     }          System.out.println(" [*] Waiting for messages. To exit press CTRL+C");     QueueingConsumer consumer = new QueueingConsumer(channel);     channel.basicConsume(queueName, true, consumer);     while (true) {       QueueingConsumer.Delivery delivery = consumer.nextDelivery();       String message = new String(delivery.getBody());       String routingKey = delivery.getEnvelope().getRoutingKey();       System.out.println(" [x] Received '" + routingKey + "':'" + message + "'");        }   } }
```

接收端和场景3的区别：

在绑定queue和exchange的时候使用了routing key，即从该exchange上只接收routing key指定的消息。

**场景5：Topics (按topic发送接收)**

使用场景：发送端不只按固定的routing key发送消息，而是按字符串“匹配”发送，接收端同样如此。

![img](https://dn-sdkcnssl.qbox.me/editor/Y3qyQxjtJ90BfWOX7pOH.png)

Producer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; public class EmitLogTopic {   private static final String EXCHANGE_NAME = "topic_logs";   public static void main(String[] argv) {     Connection connection = null;     Channel channel = null;     try {       ConnectionFactory factory = new ConnectionFactory();       factory.setHost("localhost");          connection = factory.newConnection();       channel = connection.createChannel();       channel.exchangeDeclare(EXCHANGE_NAME, "topic");       String routingKey = getRouting(argv);       String message = getMessage(argv);       channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());       System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");     }     catch  (Exception e) {       e.printStackTrace();     }     finally {       if (connection != null) {         try {           connection.close();         }         catch (Exception ignore) {}       }     }   }      private static String getRouting(String[] strings){     if (strings.length < 1)             return "anonymous.info";     return strings[0];   }   private static String getMessage(String[] strings){      if (strings.length < 2)             return "Hello World!";     return joinStrings(strings, " ", 1);   }      private static String joinStrings(String[] strings, String delimiter, int startIndex) {     int length = strings.length;     if (length == 0 ) return "";     if (length < startIndex ) return "";     StringBuilder words = new StringBuilder(strings[startIndex]);     for (int i = startIndex + 1; i < length; i++) {         words.append(delimiter).append(strings[i]);     }     return words.toString();   } }
```

发送端和场景4的区别：

1、exchange的type为topic

2、发送消息的routing key不是固定的单词，而是匹配字符串，如"*.lu.#"，*匹配一个单词，#匹配0个或多个单词。

Consumer：

```
import com.rabbitmq.client.ConnectionFactory; import com.rabbitmq.client.Connection; import com.rabbitmq.client.Channel; import com.rabbitmq.client.QueueingConsumer; public class ReceiveLogsTopic {   private static final String EXCHANGE_NAME = "topic_logs";   public static void main(String[] argv) {     Connection connection = null;     Channel channel = null;     try {       ConnectionFactory factory = new ConnectionFactory();       factory.setHost("localhost");          connection = factory.newConnection();       channel = connection.createChannel();       channel.exchangeDeclare(EXCHANGE_NAME, "topic");       String queueName = channel.queueDeclare().getQueue();         if (argv.length < 1){         System.err.println("Usage: ReceiveLogsTopic [binding_key]...");         System.exit(1);       }            for(String bindingKey : argv){             channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);       }            System.out.println(" [*] Waiting for messages. To exit press CTRL+C");       QueueingConsumer consumer = new QueueingConsumer(channel);       channel.basicConsume(queueName, true, consumer);       while (true) {         QueueingConsumer.Delivery delivery = consumer.nextDelivery();         String message = new String(delivery.getBody());         String routingKey = delivery.getEnvelope().getRoutingKey();         System.out.println(" [x] Received '" + routingKey + "':'" + message + "'");          }     }     catch  (Exception e) {       e.printStackTrace();     }     finally {       if (connection != null) {         try {           connection.close();         }         catch (Exception ignore) {}       }     }   } }
```

接收端和场景4的区别：

1、exchange的type为topic

2、接收消息的routing key不是固定的单词，而是匹配字符串。

注意点：

Topic exchange

Topic exchange is powerful and can behave like other exchanges. When a queue is bound with "#" (hash) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange. When special characters "*" (star) and "#" (hash) aren't used in bindings, the topic exchange will behave just like a direct one.



