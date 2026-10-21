+++
date = '2024-01-12T10:00:00+08:00'
draft = true
title = 'jenkins使用教程 异常'
tags = ["jenkins","使用教程","异常处理"]
series = ["使用教程"]
categories = ["CI/CD", "jenkins"]
+++

# jenkins使用教程 异常

## Docker 中进入 nginx 容器

```bash
docker exec -it nginx /bin/bash
```

PATH 环境变量：
```
/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## Maven 安装 Oracle JDBC 驱动

```bash
mvn install:install-file \
  -DgroupId=com.oracle.database.jdbc \
  -DartifactId=ojdbc11 \
  -Dversion=12.2.0.3 \
  -Dpackaging=jar \
  -Dfile=/app/maven/apache-maven-3.9.9/lib/ojdbc11-12.2.0.3.jar \
  -DgeneratePom=true
```

## 常见问题

一直git异常，打包异常。
