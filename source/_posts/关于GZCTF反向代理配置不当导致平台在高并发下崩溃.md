---
title: 关于GZCTF反向代理配置不当导致平台在高并发下崩溃
date:
  '[object Object]': null
tags:
  - Trouble Shooting
categories:
  - Trouble Shooting
abbrlink: 23988
---
<meta name="referrer" content="no-referrer"/>
## 环境

GZCTF单实例部署在`172.20.14.20`、`172.20.14.110`、`172.20.14.111`三台机器的K8s集群上，使用`LoadBalancer`对外暴露服务，`LoadBalancer IP`由`MetalLB`分得：`172.20.14.118`

## 问题描述

2024年11月16日，2024 Redrock CTF开赛，这次比赛使用了新的比赛平台[GZ::CTF - GZ::CTF Docs](https://gzctf.gzti.me/)。

由于反向代理配置不当，导致平台在刚开赛时就因为高并发而立即崩溃，报错显示错误代码429，并且平台响应很慢，无法正常刷新。

![b03f1323eec5f200d8684c9c35502050](https://gitee.com/beatrueman/images/raw/master/20250405145711209.png)

![226c63b8f33a22c1fb0d4c047c761447](https://gitee.com/beatrueman/images/raw/master/20250405145711167.png)

## 排查过程

在遇到429问题后，首先是检查了配置文件中和访问限制有关的参数配置，都适当的调大，发现并没有什么改善，没有解决问题。

![126925f6c7daad921c985e34c8d332fb](https://gitee.com/beatrueman/images/raw/master/20250405145711372.png)

GZCTF是单实例部署的，为了让流量分流，我将Pod扩容，但是单实例部署与多实例部署存在差别，并且据官方介绍单实例已经能满足大部分的一般比赛（满足2000人200题），并且是官方最为推荐的部署方式。多实例部署需要配置对象存储，否则会导致数据不一致的问题。

![img_v3_02gm_a97b367b-e1bb-4513-b921-79042d10b1cg](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)

最后在咨询开发者以及在检查日志的过程中发现了问题：

配置文件中的`ForwardedOptions`的`TrustedNetworks`和`TrustedProxies`字段使用的是默认值，反向代理并没有生效，logs里客户端的ip全是一致的，大量的相同ip可能超过了平台的访问阈值，最后平台很容易就崩溃了。

```
plaintext
[24-11-16 08:14:30.307 DBG] RequestLoggingMiddleware: [200]     7.10ms HTTP GET    /api/account/profile @ ::ffff:172.20.14.20 
[24-11-16 08:14:30.704 DBG] RequestLoggingMiddleware: [404]     0.22ms HTTP GET    /hub/user @ ::ffff:172.20.14.20 
[24-11-16 08:14:30.986 DBG] RequestLoggingMiddleware: [404]     1.32ms HTTP GET    /hub/user @ ::ffff:172.20.14.20
```

![image-20241125124424828](https://gitee.com/beatrueman/images/raw/master/20250405145711466.png)

## 解决

注意将`ForwardedOptions`的

- `TrustedNetworks`值为172.20.14.20/24
- `TrustedProxies`值为172.20.14.118，即GZCTF的`LoadBalancer IP`

整个`proxy chain`

```
plaintext
172.20.14.2（网关机）-> 172.20.14.118（LoadBalancer） -> 172.20.14.20 （master1）
                                                    -> 172.20.14.110（master2）
                                                    -> 172.20.14.111 （master3）
```