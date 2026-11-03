+++
date = '2026-10-21T10:00:00+08:00'
draft = true
title = 'docker安装mysql'
tags = ["docker","mysql","安装教程"]
series = ["安装教程"]
categories = ["数据库", "docker"]
+++

# docker安装mysql

## 1 查询版本

```bash
docker search docker.1ms.run/mysql
```

## 2 下载对应版本

```bash
docker pull mysql
```

## 3 启动mysql

```bash
docker run --name nacos-standalone \
  -e MODE=standalone \
  -e NACOS_AUTH_ENABLE=true \
  -e NACOS_AUTH_TOKEN=ZDRjYzJmY2I3YTU5NDk2ZDhhZDJlNDQyMDEzM2JiYmJiYjEyMzQ1Njc4OQ== \
  -e NACOS_AUTH_IDENTITY_KEY=your_identity_key \
  -e NACOS_AUTH_IDENTITY_VALUE=your_identity_value \
  -p 8080:8080 \
  -p 8848:8848 \
  -p 9848:9848 \
  -d nacos/nacos-server:latest
```
