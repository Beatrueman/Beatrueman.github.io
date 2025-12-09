---
title: BIND9 DNS智能解析实验
tags:
  - 服务器运维
  - DNS
categories:
  - 服务器运维
date:
  '[object Object]': null
abbrlink: 53842
---
<meta name="referrer" content="no-referrer"/>
**BIND（Berkeley Internet Name Domain）**  是全球使用最广泛的 **DNS 服务器软件**，最初由加州大学伯克利分校开发，目前由 **ISC（Internet Systems Consortium）**  维护。

BIND9 让一台服务器可以充当 DNS 解析器（递归）或权威服务器（发布域名解析记录）

# 目标

基于bind9的view功能，实现DNS的智能解析。

1. master和两个slave之间实现主从同步
2. slave-1和slave-2均提供对www.a.com的解析，其中slave-1解析www至1.1.1.1；slave-2解析www至2.2.2.2

# 环境

容器

- bind-master：172.28.0.2
- bind-slave-1：172.28.0.3
- bind-slave-2：172.28.0.4

zone：a.com

域名：www.a.com

- bind-slave-1视图：1.1.1.1
- bind-slave-2视图：2.2.2.2

权威NS：ns1.a.com（172.28.0.2）

NS：ns2.a.com（172.28.0.3、172.28.0.4）

## 目录结构

```
root@RainYun-8iZCjURn:/home/bind9-2# tree .
.
├── docker-compose.yml
├── master
│   ├── config
│   │   ├── named.conf
│   │   └── rndc.key
│   └── zones
│       ├── a.com.zone.1
│       ├── a.com.zone.2
│       └── named.log
├── slave-1
│   ├── config
│   │   └── named.conf
│   └── zones
└── slave-2
    ├── config
    │   └── named.conf
    └── zones
```

## docker-compose.yml

```
version: "3.9"

services:
  bind-master:
    image: internetsystemsconsortium/bind9:9.18
    container_name: bind-master
    restart: always
    entrypoint: /bin/sh
    tty: true
    stdin_open: true
    ports:
      - "253:53/tcp"
      - "253:53/udp"
    volumes:
      - ./master/config:/etc/bind
      - ./master/zones:/var/lib/bind
    environment:
      - TZ=Asia/Shanghai
    networks:
      bindnet:
        ipv4_address: 172.28.0.2

  bind-slave-1:
    image: internetsystemsconsortium/bind9:9.18
    container_name: bind-slave-1
    restart: always
    ports:
      - "153:53/tcp"
      - "153:53/udp"
    volumes:
      - ./slave-1/config:/etc/bind
    environment:
      - TZ=Asia/Shanghai
    depends_on:
      - bind-master
    networks:
      bindnet:
        ipv4_address: 172.28.0.3

  bind-slave-2:
    image: internetsystemsconsortium/bind9:9.18
    container_name: bind-slave-2
    restart: always
    ports:
      - "353:53/tcp"
      - "353:53/udp"
    volumes:
      - ./slave-2/config:/etc/bind
    environment:
      - TZ=Asia/Shanghai
    depends_on:
      - bind-master
    networks:
      bindnet:
        ipv4_address: 172.28.0.4


networks:
  bindnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

# 配置

## master

### 添加zone，配置主从

```
options {
	// bind工作目录，所有zone文件、日志文件都会在这个目录下找
    directory "/var/lib/bind";
	
	// bind监听的 ipv4 地址
    listen-on { any; };
    listen-on-v6 { any; };

	// 指定允许哪些客户端向该服务器发起查询（Query），对于递归DNS不要使用any
    allow-query { any; };
	
	// 是否允许递归查询
    recursion no;
	
	// 控制是否启用DNSSEC签名验证，递归服务器时适合开启，权威一般不用
    dnssec-validation no;
    
	// 隐藏bind版本号
	version "hidden";
};

// 定义 IP 源
acl "slave1" { 172.28.0.3; };
acl "slave2" { 172.28.0.4; };

// 定义视图
view "view1" {
	// 允许访问zone的IP来源
    match-clients { slave1; };
	
    zone "a.com" {
        type master;
		
		// zone file存储路径
        file "/var/lib/bind/a.com.zone.1";
        
		// 允许区域文件传输的IP地址
		allow-transfer { 172.28.0.3; };
		
		// zone file变更后主动通知 slave
        also-notify { 172.28.0.3; };
    };
};

view "view2" {
    match-clients { slave2; };

    zone "a.com" {
        type master;
        file "/var/lib/bind/a.com.zone.2";
        allow-transfer { 172.28.0.4; };
        also-notify { 172.28.0.4; };
    };
};
```

### 添加bind-slave-1视图的zone file

```
# 添加记录
;
; BIND Zone File
;
; $TTL 5 表示该DNS表项在client的缓存时间为5s
;
; Refresh 表示slave与master同步的时间
; Retry    表示slave与master断连后，多久进行重连检查
; Expire    表示slave与master断连多久后，该区域失效
; Negative Cache TTL 表示查询的数据不存在时，这个否定的回答在其他服务器的缓存时间
; @代表当前$ORIGIN（zone的根，通常是zone文件声明的域名，例如a.com）
; IN 表示 Internet class
; SOA（Start Of Authority）记录zone的元数据
; Serial很关键，用来判定zone是否更新（slave只在master的serial增大时拉取更新），通常为日期风格YYYYMMDDNN
; NS声明负责该zone的权威name server
; 末尾的.表示完整域名（绝对名称），如果没有末尾点会被当作相对名称，追加当前 $ORIGIN：ns1 → ns1.a.com.
; 常见的其他记录：CNAME、MX（邮箱），TXT（文本记录）、SRV（服务定位记录）、PTR（反向解析）


$TTL 1H
@   IN  SOA ns1.a.com. admin.a.com. (
        2025102304 ; Serial
        1H
        15M
        1W
        1H )

    IN  NS ns1.a.com.
    IN  NS ns2.a.com.

ns1 IN  A 172.28.0.2
ns2 IN  A 172.28.0.3

; slave-1 视图www对应 1.1.1.1
www IN  A 1.1.1.1
```

### 添加bind-slave-2视图的zone file

```
$TTL 1H
@   IN  SOA ns1.a.com. admin.a.com. (
        2025102305 ; Serial
        1H
        15M
        1W
        1H )

    IN  NS ns1.a.com.
    IN  NS ns2.a.com.

ns1 IN  A 172.28.0.2
ns2 IN  A 172.28.0.4
www IN  A 2.2.2.2
```

## slave

```
options {
    directory "/var/cache/bind";

    listen-on { any; };
    listen-on-v6 { any; };

    allow-query { any; };
    recursion no;

    dnssec-validation no;
    version "hidden";
};

zone "a.com" {
    type slave;
    masters { 172.28.0.2; };
    file "slaves/a.com.zone";
};

logging {
    channel default_log {
        file "/var/cache/bind/named.log" versions 3 size 5m;
        severity info;
        print-time yes;
    };
    category default { default_log; };
};
```

# 验证

## slave-1解析

```
dig @172.28.0.3 www.a.com
```

![c064f4f7cc4b84ed44ee0d7b6159bfa9](https://gitee.com/beatrueman/images/raw/master/20251208212412703.png)

## slave-2解析

```
dig @172.28.0.4 www.a.com
```

可以看到解析至2.2.2.2

![3372cd82c7940276f0af0816f6f7f8f0](https://gitee.com/beatrueman/images/raw/master/20251208212412704.png)

# 注意点

## 容器重启

实验采用`internetsystemsconsortium/bind9:9.18`​镜像，该镜像的`entrypoint`​为

```
/usr/sbin/named -u bind -f -c /etc/bind/named.conf -L /var/log/bind/default.log
```

当配置文件`named.conf`​语法出现问题或文件权限不正确时，容器会重启，且日志不会输出到标准输出上，也就是说通过docker log是看不到日志的。

‍

这时我们可以修改`entrypoint`​为`/bin/sh`​，手动进入容器进行调试，通过named命令来启动服务。

## 配置文件权限不正确

```
22-Oct-2025 12:37:11.678 ---------------------------------------------------- 22-Oct-2025 12:37:11.678 adjusted limit on open files from 1073741816 to 1048576
22-Oct-2025 12:37:11.678 found 2 CPUs, using 2 worker threads
22-Oct-2025 12:37:11.678 using 2 UDP listeners per interface
22-Oct-2025 12:37:11.682 DNSSEC algorithms: RSASHA1 NSEC3RSASHA1 RSASHA256 RSASHA512 ECDSAP256SHA256 ECDSAP384SHA384 ED25519 ED448
22-Oct-2025 12:37:11.682 DS algorithms: SHA-1 SHA-256 SHA-384
22-Oct-2025 12:37:11.682 HMAC algorithms: HMAC-MD5 HMAC-SHA1 HMAC-SHA224 HMAC-SHA256 HMAC-SHA384 HMAC-SHA512
22-Oct-2025 12:37:11.682 TKEY mode 2 support (Diffie-Hellman): yes
22-Oct-2025 12:37:11.682 TKEY mode 3 support (GSS-API): yes
22-Oct-2025 12:37:11.682 the initial working directory is '/'
22-Oct-2025 12:37:11.682 loading configuration from '/etc/bind/named.conf' 22-Oct-2025 12:37:11.686 directory '/var/lib/bind' is not writable
22-Oct-2025 12:37:11.686 /etc/bind/named.conf:2: parsing failed: permission denied
22-Oct-2025 12:37:11.686 loading configuration: permission denied
22-Oct-2025 12:37:11.686 exiting (due to fatal error)
```

调试用，最直接的方式是给777

ps：给过755，而且文件权限属于bind:bind，我在测试时还是没有写权限。

## rndc reload密钥问题

`rndc`​ 是 BIND 的远程控制工具（Remote Name Daemon Control）

可以用来让正在运行的 `named`​：

- 重新加载配置文件或 zone (`rndc reload`​)
- 重新签名 DNSSEC
- 清理缓存
- 停止/启动服务等

‍

**当修改zone file（增加或删除解析记录）、SOA序列号（Serial）需要自增1**

**然后使用rndc reload刷新配置**

如果遇到

```
/ # rndc reload a.com 
rndc: neither /etc/bind/rndc.conf nor /etc/bind/rndc.key was found
```

需要添加`rndc.key`​

```
cd /etc/bind
rndc-confgen -a -b 512
```

然后对生成的`rndc.key`​赋予权限

```
chown bind:bind /etc/bind/rndc.key
chmod 600 /etc/bind/rndc.key
```

理论赋上面的权限，但是测试时经常提示无写入权限，所以就直接777了

‍
