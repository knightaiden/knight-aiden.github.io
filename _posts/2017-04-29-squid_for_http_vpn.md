---
layout: post
title: 使用squid为Http/Https服务建立VPN
date: 2017-04-29 20:23
categories: Tech-Network
tags: squid network vpn
---

* content
{:toc}

> 前一阵接到任务，需要对接Google的某个接口，并且确认这个接口无法在国内直接访问，无奈之下，只好开始申请vpn服务器。说起来以前对VPN一直是一个衣来伸手饭来张口的态度，这次真的自己去搞，也真是个学习的过程。





## 环境准备
- 一台可以连接google的机器（别问我怎么搞……）
- squid

## 安装vpn
  1. 首先先登录到可以连接google的机器，```$ ping www.google.com``` 确保与google的网络确实没有问题。  
  2. 安装squid，这里我是用的yum安装```$ yum install squid```, 或者直接到官网下载 ：[squid官网](http://www.squid-cache.org/ )  
  3. 配置squid.conf，一般该配置文件在‘/etc/squid’这个目录下，或使用```$ whereis squid```查找。  
  4. 更改squid.conf文件，懒人做法是直接加入如下三行，注意重复问题  
```properties
# 添加
http_access allow all
http_port　3128
cache_mem 128 MB
```
  5. 启动squid，```$ service squid start``` (如果启动失败，可能需要执行```squid -z```来执行初始化)  
  6. 执行命令```$ netstat -anpt  |  grep squid```可以看到3128端口的监听状态  

## 客户端使用
  一般情况下，如果是http客户端的话，你只需要在你使用的语言和工具中将代理的ip和端口设置为刚刚配置的squid server即可，如果是使用浏览器等，道理相同，只需要在浏览器的代理设置里面将ip和端口配置好即可  
  正常情况下，配置完客户端，就可以在squid server中看到日志来，路径一般为：/var/log/squid/access.log, 日志中会有完整的访问记录。

## 配置文件详解
  刚刚对与squid.conf对配置实属懒人做法，真正线上系统还是要多考量各种因素来进行配置，下面介绍几个比较重要的配置
  - acl : 允许访问的权限，比如
    - acl all src 0.0.0.0/0.0.0.0 ：允许所有IP访问
    - acl localhost src 127.0.0.1/255.255.255.255 ：允午本机IP
    - acl Safe_ports port 80：允许安全更新的端口为80
  - http_access allow all ：允许所有人使用该代理
  - http_reply_access allow all：允许所有客户端使用该代理
  - icp_access deny all：禁止从邻居服务器缓冲内发送和接收ICP请求.
  - miss_access allow all：允许直接更新请求
  - cache_mem 128 MB：缓存大小设置

  其余相关配置请直接参阅官方文档：[官方文档](http://www.squid-cache.org/Doc/config/)
