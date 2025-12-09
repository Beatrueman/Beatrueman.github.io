---
title: Redis主从同步实验
tags:
  - 服务器运维
  - Redis
categories:
  - 服务器运维
date:
  '[object Object]': null
abbrlink: 18658
---
<meta name="referrer" content="no-referrer"/>
# 需求

* A：master，不加密（普通端口 6379）
* B：slave，从 A 同步，不加密（端口 16379），同时监听 TLS 加密端口 16380
* C：slave，从 B 同步，通过 TLS（端口 16380）

# 部署 

## 生成自签名TLS证书 使用openssl生成自签名证书 

```javascript
mkdir -p certs && cd certs

# 创建 CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=Redis-CA"

# 创建 server key 和证书签名请求（CSR）
openssl genrsa -out redis.key 2048
openssl req -new -key redis.key -out redis.csr -subj "/CN=redis-server"

# 使用 CA 签发 server 证书
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out redis.crt -days 3650 -sha256

# 权限设置
chmod 644 redis.crt redis.key ca.crt
```

CA相关证书

* `ca.key`：CA的私钥，用来签发证书（给redis.crt签名）
* `ca.crt`：CA的公钥证书，用来让客户端验证某个证书（比如redis.crt）是不是CA签发的，客户端拿redis.crt来验证时，会用这个公钥解签名。

Redis服务器相关证书

* `redis.key`：Redis 服务端的私钥，用来解密客户端发来的数据、以及在 TLS 握手时证明"我是 Redis 服务端"。
* `redis.csr`：证书签名请求（Certificate Signing Request）包含 Redis 服务器的公钥 + 服务器身份信息（域名、组织等），生成 CSR 的目的是交给 CA 签名，生成一个正式证书。
* `redis.crt`：Redis 服务器的证书

## Redis 配置文件

```javascript
port 6379
tls-port 6380
tls-cert-file /certs/redis.crt
tls-key-file /certs/redis.key
tls-ca-cert-file /certs/ca.crt
tls-auth-clients no

# 作为slave，同步redis-a
replicaof redis-a 6379
```

```javascript
# 关闭非加密 TCP 端口
port 0

tls-port 6380
tls-cert-file /certs/redis.crt
tls-key-file /certs/redis.key
tls-ca-cert-file /certs/ca.crt
# 单向TLS
tls-auth-clients no

# 从加密端口同步 redis-b
replicaof redis-b 6380
# 允许主从复制也走 TLS 加密通道
tls-replication yes
```

## docker-compose

启动三个redis容器

redis-a作为master，暴露容器端口6379到6379

redis-b作为slave，暴露容器端口6379到16379，容器端口6380到16380（TLS）

redis-c作为slave，暴露容器端口6380到26380（TLS）

```javascript
version: '3.8'

services:
  redis-a:
    image: redis:latest
    container_name: redis-a
    ports:
      - "6379:6379"
    volumes:
      - ./data/redis-a:/data
    command: ["redis-server", "--port", "6379"]
    networks:
      - redis-net

  redis-b:
    image: redis:latest
    container_name: redis-b
    ports:
      - "16379:6379"
      - "16380:6380"
    volumes:
      - ./data/redis-b:/data
      - ./certs:/certs
      - ./redis-b.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-net
    depends_on:
      - redis-a

  redis-c:
    image: redis:latest
    container_name: redis-c
    ports:
      - "26380:6380"
    volumes:
      - ./data/redis-c:/data
      - ./certs:/certs
      - ./redis-b.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - redis-net
    depends_on:
      - redis-b


networks:
  redis-net:
    driver: bridge
```

# 验证

## 查看主从状态

目标：

* B 是 A 的从节点
* C 是 B 的从节点（通过 TLS）

### 查看 Redis B 是否同步 Redis A（非 TLS）

可以看到b以a作为master，`master_link_status:up` 说明同步正常

```javascript
root@liyixiong01:~/redis# docker exec -it redis-b redis-cli -p 6379 info replication
# Replication
role:slave
master_host:redis-a
master_port:6379
master_link_status:up
```

### 查看 Redis C 是否通过 TLS 同步 Redis B

```javascript
root@liyixiong01:~/redis# docker exec -it redis-c redis-cli   --tls   --cert /certs/redis.crt   --key /certs/redis.key   --cacert /certs/ca.crt   -p 6380 info replication
# Replication
role:slave
master_host:redis-b
master_port:6380
master_link_status:up
```

## 验证数据同步链路是否真实工作

### 在 Redis A 写入数据

```javascript
root@liyixiong01:~/redis# docker exec -it redis-a redis-cli -p 6379 set hello world
OK
```

### 在 Redis B 查看是否能读到（非TLS）

```javascript
root@liyixiong01:~/redis# docker exec -it redis-b redis-cli -p 6379 get hello
\"world"
```

### 在 Redis C 查看是否能读到（TLS）

```javascript
root@liyixiong01:~/redis# docker exec -it redis-c redis-cli \
  --tls \
  --cert /certs/redis.crt \
  --key /certs/redis.key \
  --cacert /certs/ca.crt \
  -p 6380 get hello
"world"
```