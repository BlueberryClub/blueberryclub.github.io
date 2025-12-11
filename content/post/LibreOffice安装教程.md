+++
date = '2025-12-11T10:30:52+08:00'
draft = true
title = 'LibreOffice安装教程'
tags = ["教程","libreOffice"]
series = ["教程"]
categories = ["libreOffice"]
+++
2. 使用文件上传工具将libreOffice 上传至服务器

![Image Alt Text](/images/LibreOffice安装教程1.png)

2.  将文件解压

![Image Alt Text](/images/LibreOffice安装教程2.png)

3. 进行软件的安装   **rpm -ivh *.rpm**

![Image Alt Text](/images/LibreOffice安装教程3.png)

4. 配置项目的路径

这个是打开配置文件的命令和使配置文件生效的命令

![Image Alt Text](/images/LibreOffice安装教程4.png)

![Image Alt Text](/images/LibreOffice安装教程5.png)


5. 执行报错

![Image Alt Text](/images/LibreOffice安装教程6.png)

6. 安装依赖

<font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);">yum install alsa-lib-devel</font>

<font style="color:rgb(0, 0, 0);background-color:rgb(250, 250, 250);">yum install gtk2</font>

7. 安装完成

![Image Alt Text](/images/LibreOffice安装教程7.png)



因为没有出现乱码，所以未使用两个中文插件包

教程和上面的几乎一致   <font style="color:rgb(79, 79, 79);">rpm -Uvh *.rpm</font>

![Image Alt Text](/images/LibreOffice安装教程8.png)

<font style="color:rgb(51, 51, 51);">bashrc    、 profile 两个配置文件</font>

