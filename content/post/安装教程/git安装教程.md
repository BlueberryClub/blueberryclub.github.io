+++
date = '2024-01-15T10:00:00+08:00'
draft = true
title = 'git安装教程'
tags = ["git","安装教程"]
series = ["安装教程"]
categories = ["版本控制", "git"]
+++

# git安装教程

使用 Docker 安装 GitLab：

```bash
docker run -d \
  -p 8443:443 \
  -p 8090:80 \
  -p 8022:22 \
  --restart always \
  --name gitlab \
  -v /usr/local/gitlab/etc:/etc/gitlab \
  -v /usr/local/gitlab/log:/var/log/gitlab \
  -v /usr/local/gitlab/data:/var/opt/gitlab \
  --privileged=true gitlab/gitlab-ce
```
