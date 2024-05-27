---
author: ["柿子"]
title: '如何构建RocketMQ-5.2源码调试环境'
date: 2024-05-24T13:06:57+08:00
summary: "构建最小可调试环境"
tags: ["RocketMQ","Debug","源码"]
typora-root-url: ../../static
typora-copy-images-to: ../../static/images
---

## 前言

源于看到了这两篇博客[基于 Rust 的高性能 RocketMQ Proxy](https://juejin.cn/post/7321551188092911668)、[RocketMQ 5.0：无状态代理模式的探索与实践](https://www.cnblogs.com/alisystemsoftware/p/16776559.html)，兴趣使然，想探究下Proxy现有的设计实现，为自主实现各大MQ协议互转Proxy代理铺垫。也就是第一篇博客实现的Proxy，看起来在RocketMQ未来规划中是有的。关于Coding需要体会下才得道。

![1.png](/images/0cf0153cb97c4df5bb0b31e006c7296e~tplv-k3u1fbpfcp-zoom-1.png)

下面开始。。。



从0到“Hello World”，最快的方式优先从“[QuickSstart](https://rocketmq.apache.org/zh/docs/quickStart/01quickstart)”体验下。体验下总结就是这麽几个步骤：

1. 启动NameServer
2. 选择``Broker+Proxy`同进程，`Broker`、`Proxy`分离(不同进程)部署，不过内部的步骤也是先启动`Broker`，再`Proxy`
3. 测试`Producer`/`Consumer`



总所周知，源码不会唬人，所以接下来基于源码按照这个步骤展开。。。。

> 这里用的IDE主要是 Idea。

## 克隆项目

```bash
# 克隆代码
git clone https://github.com/apache/rocketmq.git
```

- 切换到 release-2.5.0

- 构建Maven依赖

- 然后IDE中maven install或者终端`mvn clean install -Dmaven.test.skip=true`

  ![image-20240524下午20029614](/images/image-20240524%E4%B8%8B%E5%8D%8820029614.png)

  - 最后主要关注项目中中这几个模块

    ![image-20240524下午20550779](/images/image-20240524%E4%B8%8B%E5%8D%8820550779.png)

## 启动`NameServer`

1. 复制“distribution”模块的绝对路径

2. IDE-配置“NameServStartup”启动项

   1. 找到“namasrv”模块的启动类“NameServStartup”

   2. main方法启动，然后会提示需要设置“ROCKETMQ_HOME”

   3. 打开启动配置框

      ![image-20240527下午11922858](/images/image-20240527%E4%B8%8B%E5%8D%8811922858.png)

   4. 设置环境变量“ROCKETMQ_HOME”为“distribution”模块绝对路径，确定

3. 重新启动，看下面打印，启动成功

   ![image-20240527下午12103203](/images/image-20240527%E4%B8%8B%E5%8D%8812103203.png)

## 选择 `Broker`、`Proxy`分离部署

选择分离部署，字面意思也是互不影响，主要是方便调试，因为我们的目标是`Proxy`。

### 启动Broker

1. 找到“broker”模块下的“BrokerStartup”启动类，main函数启动（会提示没有“ROCKETMQ_HOME”）

2. 打开IDE启动配置，设置启动参数和环境变量“ROCKETMQ_HOME”

   ![image-20240527下午33302390](/images/image-20240527%E4%B8%8B%E5%8D%8833302390.png)

   添加VM参数：-Xms512m -Xmx512m -Xmn256m

   -n 是你本机的nameserv，把“192.168.1.128”改成你自己的

   设置环境变量“ROCKETMQ_HOME”为之前复制的“distribution”的绝对路径

3. 重新启动，启动打印如下

   ![image-20240527下午33445533](/images/image-20240527%E4%B8%8B%E5%8D%8833445533.png)

### 启动Proxy

1. 找到“proxy”模块中启动类“ProxyStartup”，main函数启动，然后停止

2. 打开启动配置，设置启动参数和环境变量

   ![image-20240527下午33921431](/images/image-20240527%E4%B8%8B%E5%8D%8833921431.png)

   -n “192.168.1.128”修改为你的IP，表示代理的broker-server服务

3. 重新启动，打印如下

   ![image-20240527下午34020223](/images/image-20240527%E4%B8%8B%E5%8D%8834020223.png)

## 测试

### 直连`broker`

- `Producer`

  1. 找到“example”模块“quickstart”包下“Producer”类，main启动（会报错连接不上），停止

  2. 打开启动配置，设置环境变量：NAMESRV_ADDR

     ![image-20240527下午35612583](/images/image-20240527%E4%B8%8B%E5%8D%8835612583.png)

     修改“192.168.1.128”为你的IP

  3. 重新启动，推送消息打印

     ![image-20240527下午35929528](/images/image-20240527%E4%B8%8B%E5%8D%8835929528.png)

  4. 异常如果是下面截图所示，暂时快速解决方法：重启 BrokerStartup

     ![image-20240527下午40429418](/images/image-20240527%E4%B8%8B%E5%8D%8840429418.png)

- `Consumer`

  1. 找到“example”模块“quickstart”包下“Consumer”类，main启动（会报错连接不上），停止

  2. 打开启动配置，设置环境变量：NAMESRV_ADDR

     ![image-20240527下午40726616](/images/image-20240527%E4%B8%8B%E5%8D%8840726616.png)

  3. 重启启动，消费记录如下

     ![image-20240527下午40702811](/images/image-20240527%E4%B8%8B%E5%8D%8840702811.png)

### 通过`proxy`测试

根据官方[QuickStart中SDK测试消息收发](https://rocketmq.apache.org/zh/docs/quickStart/01quickstart#5-sdk测试消息收发)指南进行测试。

1. 创建Java工程

2. 引入maven依赖，查看[最新版本](https://search.maven.org/search?q=g:org.apache.rocketmq%20AND%20a:rocketmq-client-java)

   ```java
   <dependency>
       <groupId>org.apache.rocketmq</groupId>
       <artifactId>rocketmq-client-java</artifactId>
       <version>${rocketmq-client-java-version}</version>
   </dependency> 
   ```

3. 通过mqadmin创建 Topic。

   这里指南中其实使用的`Rocketmq源码`中的`tools`模块下``MQAdminStartup`类，所以接下来我们回到源码工程中：

   1. 启动MQAdminStartup类，看结果，是一个命令行工具

      <img src="/images/image-20240527%E4%B8%8B%E5%8D%8842039785.png" alt="image-20240527下午42039785" style="zoom:50%;" />

   2. 打开启动配置，添加启动参数：`updatetopic -n localhost:9876 -t TestTopic -c DefaultCluster`

      <img src="/images/image-20240527%E4%B8%8B%E5%8D%8842237990.png" alt="image-20240527下午42237990" style="zoom:50%;" />

   3. 执行成功如下打印

      <img src="/images/image-20240527%E4%B8%8B%E5%8D%8842309734.png" alt="image-20240527下午42309734" style="zoom:50%;" />

   4. 出此之外你还可以安装：[apache/rocketmq-dashboard](https://github.com/apache/rocketmq-dashboard)，用来可视化查看mq的一些情况，建议使用docker安装，`注意web端口8080映射为别的端口，默认和proxy默认端口冲突了`

4. `Producer`：回到刚刚的新工程中，创建“ProducerExample”类

   ```java
   import org.apache.rocketmq.client.apis.ClientConfiguration;
   import org.apache.rocketmq.client.apis.ClientConfigurationBuilder;
   import org.apache.rocketmq.client.apis.ClientException;
   import org.apache.rocketmq.client.apis.ClientServiceProvider;
   import org.apache.rocketmq.client.apis.message.Message;
   import org.apache.rocketmq.client.apis.producer.Producer;
   import org.apache.rocketmq.client.apis.producer.SendReceipt;
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   
   public class ProducerExample {
       private static final Logger logger = LoggerFactory.getLogger(ProducerExample.class);
   
       public static void main(String[] args) throws ClientException {
           // 接入点地址，需要设置成Proxy的地址和端口列表，一般是xxx:8081;xxx:8081。
           String endpoint = "192.168.1.128:8081";
           // 消息发送的目标Topic名称，需要提前创建。
           String topic = "TestTopic";
           ClientServiceProvider provider = ClientServiceProvider.loadService();
           ClientConfigurationBuilder builder = ClientConfiguration.newBuilder().setEndpoints(endpoint);
           ClientConfiguration configuration = builder.build();
           // 初始化Producer时需要设置通信配置以及预绑定的Topic。
           Producer producer = provider.newProducerBuilder()
               .setTopics(topic)
               .setClientConfiguration(configuration)
               .build();
           // 普通消息发送。
           Message message = provider.newMessageBuilder()
               .setTopic(topic)
               // 设置消息索引键，可根据关键字精确查找某条消息。
               .setKeys("messageKey")
               // 设置消息Tag，用于消费端根据指定Tag过滤消息。
               .setTag("messageTag")
               // 消息体。
               .setBody("messageBody".getBytes())
               .build();
           try {
               // 发送消息，需要关注发送结果，并捕获失败等异常。
               SendReceipt sendReceipt = producer.send(message);
               logger.info("Send message successfully, messageId={}", sendReceipt.getMessageId());
           } catch (ClientException e) {
               logger.error("Failed to send message", e);
           }
           // producer.close();
       }
   }
   ```

5. `Consumer`：创建“PushConsumerExample”类

   ```java
   import java.io.IOException;
   import java.util.Collections;
   import org.apache.rocketmq.client.apis.ClientConfiguration;
   import org.apache.rocketmq.client.apis.ClientException;
   import org.apache.rocketmq.client.apis.ClientServiceProvider;
   import org.apache.rocketmq.client.apis.consumer.ConsumeResult;
   import org.apache.rocketmq.client.apis.consumer.FilterExpression;
   import org.apache.rocketmq.client.apis.consumer.FilterExpressionType;
   import org.apache.rocketmq.client.apis.consumer.PushConsumer;
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   
   public class PushConsumerExample {
       private static final Logger logger = LoggerFactory.getLogger(PushConsumerExample.class);
   
       private PushConsumerExample() {
       }
   
       public static void main(String[] args) throws ClientException, IOException, InterruptedException {
           final ClientServiceProvider provider = ClientServiceProvider.loadService();
           // 接入点地址，需要设置成Proxy的地址和端口列表，一般是xxx:8081;xxx:8081。
           String endpoints = "192.168.1.128:8081";
           ClientConfiguration clientConfiguration = ClientConfiguration.newBuilder()
               .setEndpoints(endpoints)
               .build();
           // 订阅消息的过滤规则，表示订阅所有Tag的消息。
           String tag = "*";
           FilterExpression filterExpression = new FilterExpression(tag, FilterExpressionType.TAG);
           // 为消费者指定所属的消费者分组，Group需要提前创建。
           String consumerGroup = "YourConsumerGroup";
           // 指定需要订阅哪个目标Topic，Topic需要提前创建。
           String topic = "TestTopic";
           // 初始化PushConsumer，需要绑定消费者分组ConsumerGroup、通信参数以及订阅关系。
           PushConsumer pushConsumer = provider.newPushConsumerBuilder()
               .setClientConfiguration(clientConfiguration)
               // 设置消费者分组。
               .setConsumerGroup(consumerGroup)
               // 设置预绑定的订阅关系。
               .setSubscriptionExpressions(Collections.singletonMap(topic, filterExpression))
               // 设置消费监听器。
               .setMessageListener(messageView -> {
                   // 处理消息并返回消费结果。
                   logger.info("Consume message successfully, messageId={}", messageView.getMessageId());
                   return ConsumeResult.SUCCESS;
               })
               .build();
           Thread.sleep(Long.MAX_VALUE);
           // 如果不需要再使用 PushConsumer，可关闭该实例。
           // pushConsumer.close();
       }
   }
   ```

6. 启动`Producer`，打印如下

   ![image-20240527下午45229672](/images/image-20240527%E4%B8%8B%E5%8D%8845229672.png)

7. 启动`Consumer`，打印如下

   ![image-20240527下午45312605](/images/image-20240527%E4%B8%8B%E5%8D%8845312605.png)

---

好了，debug环境搭建好了，可以开始愉快的Debug。。。
