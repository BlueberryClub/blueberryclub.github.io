+++
date = '2025-12-11T08:31:52+08:00'
draft = true
title = 'Kafka安装教程'
tags = ["Kafka","安装教程"]
series = ["安装教程"]
categories = ["中间件", "Kafka"] 
+++



## 软件下载
因为需要做数据接入功能的开发，所以使用到了kafka的工具。不过测试不好测试，所以就在远程服务器安装一个kafka，来进行测试，故记录此次安装过程为以后做积累。

:::tips
1. 需要安装 si
2. 将程序包放入服务器上。(程序包很好搞)

:::

## 软件安装
:::tips
1. 将压缩包解压

tar -xzf kafka_2.11-0.11.0.2.tgz

2. 进入文件下，修改  vi config/server.properties

![Image Alt Text](/images/Kafka安装教程1.png)

修改这个的  <font style="color:rgb(0, 0, 0);">zookeeper.connect=192.168.1.202:2181</font>

修改  vi bin/kafka-run-class.sh 去查看问题，因为java的路径是分块安装的。

3. bin/kafka-server-start.sh config/server.properties 启动kafka
4.  去  /lib/systemd/system 下面新建一个 kafka.service 将命令放入

[Unit]

Description=kafkaservice

After=network.target



[Service]

WorkingDirectory=/soft/KafKa/kafka_2.11-0.11.0.2

ExecStart=/soft/KafKa/kafka_2.11-0.11.0.2/bin/kafka-server-start.sh config/server.properties

ExecStop=/soft/KafKa/kafka_2.11-0.11.0.2/bin/kafka-server-stop.sh

User=root

Group=root

Restart=always

RestartSec=10



[Install]

WantedBy=multi-user.target

4.  让自启动生效   systemctl enable kafka.service
5.  立即启动服务   systemctl start kafka.service
6. 查看启动状态    systemctl status kafka.service

:::





