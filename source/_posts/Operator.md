---
title: Operator介绍
date:
  '[object Object]': null
tags:
  - Kubernetes
  - Operator
categories:
  - Kubernetes
abbrlink: 58647
---
<meta name="referrer" content="no-referrer"/>
# Operator

## 为什么要用？

当K8s中的原生资源无法满足业务要求时，这时候就需要Operator来创建自己的CRD以及对应Controller。利用Controller实现CRD所需要的业务逻辑。

比如，我们想在Kubernetes中部署一个Nginx并让外界可以访问到，这时我们首先需要创建一个Deployment用来部署Nginx，然后创建一个Service用来将Deployment的Pod的容器端口映射到主机端口。我们觉得太麻烦了，想要创建一次资源就可以同时包含Deployment和Service。

## 是什么？

Operator是一种**通过自定义控制器（Customer Controller）扩展K8s API的模式**。它的核心思想是将运维知识代码化。允许开发者将应用的管理逻辑（如部署、升级、备份、恢复）封装成代码，使K8s能够像管理原生资源一样管理复杂的有状态应用（如数据库、消息队列）

## 解决了什么？

- 管理有状态应用的复杂性：例如pg集群的自动故障转移、redis的数据持久化
- 自动化运维：通过代码替代人工操作
- 统一生命周期管理：标准化应用的安装、配置、扩缩容流程等

## 什么时候用？

- 数据库管理：MariaDB Operator
- 中间件管理：Kafka Operator
- CI/CD工具：Argo CD Operator

## 核心组件

### Customer Resource Definitions（CRD）

CRD是K8s中扩展API资源的方式。

### Controller

- 职责：监听资源状态变化、驱动集群向期望状态收敛
- 核心逻辑：通过Reconcile()函数实现调谐循环

### Reconcile Loop

调谐循环机制

- 获取资源的当前状态
- 对比实际状态（集群中运行的Pod数量）
- 执行操作（增减Pod）
- 更新资源状态（status字段）

## 和Helm对比？

区别与场景

- Helm：适合静态配置的包管理工具（如一次性部署应用）
- Operator：适合的动态、持续管理的场景（如自动修复故障节点）
