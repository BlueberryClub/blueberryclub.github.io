+++
date = '2025-12-11T10:31:52+08:00'
draft = true
title = 'rocketMq安装教程'
tags = ["教程","中间件"]
series = ["教程"]
categories = ["mq"]
+++
# rocketMq安装和使用
##  软件下载
### 下载与上传
因为是使用的腾讯云服务器，所有有内网ip和外网ip两个ip，所以这个需要注意。我使用的是本地下载然后将文件上传至服务器的方式进行处理的。文件如下：

![](https://cdn.nlark.com/yuque/0/2023/png/29678232/1678344809582-7a1d65d7-98aa-458b-93dd-8b867573e5d7.png)

## 软件安装
将文件解压  unzip + 文件名，将文件解压后内容如下，结构和目录非常清晰。

![](https://cdn.nlark.com/yuque/0/2023/png/29678232/1678344961776-d7df6d7b-3a9b-480b-a775-4cd5bce154d3.png)

bin文件和conf文件是我们要重点关注的东西，readme是了解这个下东西的基本介绍。

```java
// 1. 启动项目的 nameServer
nohup ./bin/mqnamesrv &
// 1.1 测试 nameSrv 启动是否正常
netstat -an | grep 9876
// 2. 启动项目的 broker 
nohup sh bin/mqbroker -c conf/broker.conf autoCreateTopicEnable=true &
// 2.1 测试 broker 启动是否正常
tail -f ~/logs/rocketmqlogs/broker.log
```

1.1 正确结果展示

![](https://cdn.nlark.com/yuque/0/2023/png/29678232/1678345976086-7e2f564e-b30c-4650-a0e1-4740af98a6d0.png)

2.1 正确结果展示  重点是那个ip 是外网ip

![](https://cdn.nlark.com/yuque/0/2023/png/29678232/1678351824063-f0b3bb6b-a5fd-4bc0-91ab-5700ce2c20b1.png)

## <font style="color:rgb(0, 0, 0);">软件测试</font>
消费者

```java
package com.xy.Utils;

//import com.alibaba.fastjson.JSONObject;

import org.apache.rocketmq.acl.common.AclClientRPCHook;
import org.apache.rocketmq.acl.common.SessionCredentials;
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.consumer.rebalance.AllocateMessageQueueAveragely;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.message.MessageExt;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@Configuration
@EnableScheduling
@EnableAsync
public class Task {

    private static String producerGroup = "DefaultCluster";

    private static String namesrvAddr = "124.222.116.241:9876";
    private static String accessKey = "test";
    private static String secretKey = "test";


    public static void main(String[] args) throws MQClientException {
        //创建消息消费者
        // DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(producerGroup);
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(producerGroup,
                new AclClientRPCHook(new SessionCredentials(accessKey, secretKey)), new AllocateMessageQueueAveragely());

        //设置NameServer
        consumer.setNamesrvAddr(namesrvAddr);
        //指定订阅的主题和标签
        consumer.subscribe("TopicTest","*");
        //回调函数
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                System.out.println(list);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        //启动消费者
        consumer.start();
        System.out.println("Consumer Started .%n");
    }


//    @Async
//    @Scheduled(cron = "0 */20 * * * ?")
//    public void rocketMqTask() throws MQClientException {
//        // Instantiate with specified consumer group name.
//        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
//        // Specify name server addresses.
//        consumer.setNamesrvAddr("10.92.190.36:80");
//        // Subscribe one more more topics to consume.
//        consumer.subscribe("0301", "*");
//        // Register callback to execute on arrival of messages fetched from brokers.
//        consumer.registerMessageListener(new MessageListenerConcurrently() {
//
//            @Override
//            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
//                                                            ConsumeConcurrentlyContext context) {
//                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
//                // 1. 解析数据，获取到 工单id、编号、区局编码
//
//
//                // 2. 使用工单id、编号、区局编码获取故障抢修数据     需要: orderId
//                String str = "";
//                String url = "";
//                String date = "";
//                str = uploadUtil.uploadFile(url, date);
////                JSONObject obj1 = JSONObject.parseObject(str.toString());
////                if (obj1.equals(null) || obj1.isEmpty()){
////                    // 未查询到任何数据
////                    System.out.println("");
////                    return null;
////                }else {
////
////
////                }
//                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
//            }
//        });
//        //Launch the consumer instance.
//        consumer.start();
//        System.out.printf("Consumer Started.%n");
//    }

}

```

生产者



```java
package com.xy.Utils;

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

public class MsgProducer {

    private static String producerGroup = "DefaultCluster";

    private static String namesrvAddr = "124.222.116.241:9876";

    public static void main(String[] args) throws Exception {
        //创建消息生产者
        DefaultMQProducer producer = new DefaultMQProducer(producerGroup);
        //设置NameServer
        producer.setNamesrvAddr(namesrvAddr);
        //启动生产者
        producer.start();
        // 创建主题
//        producer.createTopic("broker-a","myTopic",4);
        producer.setSendMsgTimeout(15000);
        //构建消息对象
//        Message message = new Message("myTopic","myTag",("Test MQ").getBytes());
        Message message = new Message("TopicTest","myTag",("好诡异").getBytes());
        producer.setSendMsgTimeout(15000);
        //发送消息

        SendResult result = producer.send(message, 10000);
        producer.setSendMsgTimeout(15000);
        System.out.println(result);
        //关闭生产者
        producer.shutdown();
    }



}

```

