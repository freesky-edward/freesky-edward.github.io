---
layout: post
title: "使用shadowsocks访问google搜索"
date: 2018-05-26
lang: zh
---

### 简要说明

要做到访问像google这样的账号，有几种方式可以选择

1. 购买VPN
2.  自己搭建一套代理服务器

本文是基于第二种，但是不会介绍如何搭建（如需要请参考 [这里](../build-shadowsocks-with-internal) ），这里只是介绍本地如何使用，客户端使用体验账号。

### 安装环境准备

安装代理客户端，软件包下载地址：

> android: https://github.com/shadowsocks/shadowsocks-android/releases     
> windows： https://github.com/shadowsocks/shadowsocks-windows/releases    
> iphone： https://github.com/shadowsocks/shadowsocks-iOS/releases    

具体的安装过程这里不再详细阐述，

### 创建体验账号

如果有自己的代理服务器，请略过本步。如果没有请参考创建体验账号。特别说明，由于是体验账号，创建后如需长期使用请告知作者，联系方式见下。

1. 登录 [http://newto.me:9080](http://newto.me:9080) 注册用户。注册时只需要验证邮箱，无其他步骤。
2. 登录系统获取体验账号信息。登录后点击左侧【账号】记录下如下信息：


    服务器地址： newto.me 注意：不是界面显示地址    
    端口： 8888  注：在中间页面上方    
    登录密码： *****     
    加密方式：aes-256-cfb    

### 使用代理

打开第一步创建的客户端，添加一个服务器，依次输入第二步记录的信息后，启动连接即可。

由于不同客户端操作方式不同，这里就不一一详解了，操作相对比较简单，摸索下即可完成。

### FAQ

Q： 创建账号后配置客户端不能访问？    
A： 刚创建账号并不是立马生效，生效时间有一定的延迟，如果不能访问稍微等5-10分钟后再尝试。


