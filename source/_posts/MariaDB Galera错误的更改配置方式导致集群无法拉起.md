---
title: MariaDB Galera错误的更改配置方式导致集群无法拉起
date:
  '[object Object]': null
tags:
  - Trouble Shooting
categories:
  - Trouble Shooting
abbrlink: 52501
---

<meta name="referrer" content="no-referrer"/>

# 故障描述

`sre-tools-database` 命名空间下由 **MariaDB Operator 管理的 Galera 集群**包含 3 个 Pod 实例。在一次配置变更过程中，将对应的 **StatefulSet 直接缩容至 0**，随后再扩容回 3。

扩容后，三个 Pod 均无法进入 `Running` 状态，集群节点相互等待（死锁），始终无法形成 **Primary View**，最终因连接超时而启动失败。

```bash
mariadb-galera-0日志
...
2025-12-21 14:10:02 0 [Note] WSREP: gcomm: connecting to group 'mariadb-operator', peer 'mariadb-galera-0.mariadb-galera-internal.sre-tools-database.svc.cluster.local:,mariadb-galera-1.mariadb-galera-internal.sre-tools-database.svc.cluster.local:,mariadb-galera-2.mariadb-galera-internal.sre-tools-database.svc.cluster.local:'
2025-12-21 14:10:02 0 [Note] WSREP: (be2303ad-b24e, 'tcp://0.0.0.0:4567') Found matching local endpoint for a connection, blacklisting address tcp://100.112.98.78:4567
2025-12-21 14:10:05 0 [Note] WSREP: EVS version upgrade 0 -> 1
2025-12-21 14:10:05 0 [Note] WSREP: PC protocol upgrade 0 -> 1
[*]2025-12-21 14:10:05 0 [Warning] WSREP: no nodes coming from prim view, prim not possible
[*]2025-12-21 14:10:05 0 [Note] WSREP: view(view_id(NON_PRIM,be2303ad-b24e,1) memb {
be2303ad-b24e,0
} joined {
} left {
} partitioned {
})
2025-12-21 14:10:06 0 [Warning] WSREP: last inactive check more than PT1.5S ago (PT3.50146S), skipping check
2025-12-21 14:10:35 0 [Note] WSREP: PC protocol downgrade 1 -> 0
2025-12-21 14:10:35 0 [Note] WSREP: view((empty))
[*]2025-12-21 14:10:35 0 [ERROR] WSREP: failed to open gcomm backend connection: 110: failed to reach primary view: 110 (Connection timed out)
at ./gcomm/src/pc.cpp:connect():160
2025-12-21 14:10:35 0 [ERROR] WSREP: ./gcs/src/gcs_core.cpp:gcs_core_open():221: Failed to open backend connection: -110 (Connection timed out)
2025-12-21 14:10:36 0 [ERROR] WSREP: ./gcs/src/gcs.cpp:gcs_open():1674: Failed to open channel 'mariadb-operator' at 'gcomm://mariadb-galera-0.mariadb-galera-internal.sre-tools-database.svc.cluster.local,mariadb-galera-1.mariadb-galera-internal.sre-tools-database.svc.cluster.local,mariadb-galera-2.mariadb-galera-internal.sre-tools-database.svc.cluster.local': -110 (Connection timed out)
[*]2025-12-21 14:10:36 0 [ERROR] WSREP: gcs connect failed: Connection timed out
2025-12-21 14:10:36 0 [ERROR] WSREP: wsrep::connect(gcomm://mariadb-galera-0.mariadb-galera-internal.sre-tools-database.svc.cluster.local,mariadb-galera-1.mariadb-galera-internal.sre-tools-database.svc.cluster.local,mariadb-galera-2.mariadb-galera-internal.sre-tools-database.svc.cluster.local) failed: 7
[*]2025-12-21 14:10:36 0 [ERROR] Aborting
root@Redrock-ButterBeer:/var/lib/mysql#
```

# 故障原因分析

## **直接原因：**不当的运维操作导致 Galera 集群状态完全丢失

在修改数据库配置时，直接编辑了下游的 `ConfigMap`，并试图通过 **将 StatefulSet 缩容至 0 再扩容** 的方式强制使配置生效。

该操作等同于同时下线所有Galera节点，即终止集群。

> @[https://mariadb.com/docs/galera-cluster/galera-management/installation-and-deployment/getting-started-with-mariadb-galera-cluster#restarting-the-cluster](mention://0f8566e2-d4e6-497d-8e2f-45fb4890a233/url/d32a8931-d64d-45c8-a442-3c3e74ae5cc4) 
>
> If you shut down all nodes at the same time, then you have effectively terminated the cluster. Of course, the cluster's data still exists, but the running cluster no longer exists. When this happens, you'll need to bootstrap the cluster again.

导致

* 集群中不存在任何存活节点可作为 **Primary Component**
* 没有节点持有可用于恢复的有效集群视图（View）
* 所有节点在启动时都在等待其他节点先行加入，形成"相互等待"的死锁状态

最终集群无法自动选主，全部节点启动失败。

## **深层原因**


1. **绕过了Operator的声明式管理逻辑：**没有意识到由Operator管理的资源，应该通过修改上层CR的Spec来触发Operator的Reconcile逻辑，而不是直接修改干预下层资源（ConfigMap、StatefulSet），导致 Operator 无法自动识别集群的故障状态并触发内置的故障恢复逻辑。
2. **缺乏测试验证：**进行高风险破环性更新（即同时停止所有数据库实例）前应该先测试，违反了运维操作规范。不过幸好有备份。
3. **容灾建设不完善：**没有集群恢复的标准SOP，对于故障发生后也只能依赖临时排障+文档查阅，恢复成本高，时间周期长。

# 故障发生时间线

* 2025/12/21 21:40

  由于 **SRE_center** 数据库的 **change_info** 表在插入中文数据后显示为拉丁字符，排查发现 `character-set-server` 为 `latin`，因此决定调整为 `utf8mb4`。\n通过以下命令修改 MariaDB Galera 的配置 ConfigMap：

  `kubectl edit cm -n sre-tools-database mariadb-galera-config `

  新增配置内容：

  ```bash
  [mariadb]
  character-set-server=utf8mb4 
  collation-server=utf8mb4_general_ci 
  ```


---

* 2025/12/21 21:50

  修改配置后，对 `mariadb-galera` StatefulSet 进行操作，**直接缩容至 0，再扩容至 3**。\n新启动的 Pod 无法正常进入 `Running` 状态，Pod 日志中报错（详见故障描述）。\n3 个 Pod 无法完成选主，集群进入死锁状态。


---

* 2025/12/21 22:00

  查阅官方文档及相关资料后确认：\n当 Galera 集群被整体终止后，需要人工介入重新引导集群。

  恢复思路为：
  * 先启动一个节点作为 **Most Advanced Node**
  * 再由其他节点加入该集群

  因此将 StatefulSet 缩容至 1，尝试启动 Pod 0。


---

* 2025/12/21 23:44

  操作时先暂停Operator对MariaDB集群的管理

  ```bash
  kubectl patch mariadb mariadb-galera -n sre-tools-database \
    --type merge -p '{"spec":{"suspend":true}}'
  ```

  通过修改 Pod 0 对应 PV 中 `/var/lib/mysql/grastate.dat` 文件，将 `safe_to_bootstrap` 设置为 `1`，并在 StatefulSet 中为 mariadb 容器添加启动参数 `mariadbd --wsrep_new_cluster`，成功使 Pod 0 正常启动。

  ```bash
  # GALERA saved state
  version: 2.1
  uuid:    0a9026bb-de80-11f0-aa0c-7ec0fff494fd
  seqno:   -1
  safe_to_bootstrap: 1
  ```


---

* 2025/12/21 23:50

  对 Pod 1、Pod 2 也采用了相同方式进行启动。\n但该操作会导致每个 Pod 都被视为独立集群，形成集群脑裂，不符合主从架构。

  ```bash
  MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_cluster_size';
  +--------------------+-------+
  | Variable_name      | Value |
  +--------------------+-------+
  | wsrep_cluster_size | 1     |
  +--------------------+-------+
  1 row in set (0.001 sec)
  MariaDB [(none)]>
  ```


---

* 2025/12/22 00:01

  删除 Pod 1、Pod 2 对应 PV 中的 `grastate.dat` 文件，\n并移除 StatefulSet 中的 `--wsrep_new_cluster` 启动参数，随后重新启动这两个 Pod。

  Pod 正常 `Running` 后，检查集群状态：

  ```bash
  MariaDB [(none)]> SHOW STATUS LIKE 'wsrep_cluster_size';
  +--------------------+-------+
  | Variable_name      | Value |
  +--------------------+-------+
  | wsrep_cluster_size | 3     |
  +--------------------+-------+
  1 row in set (0.001 sec)
  MariaDB [(none)]>
  ```

  恢复Operator对MariaDB的管理

  ```bash
  kubectl patch mariadb mariadb-galera -n sre-tools-database \
      --type merge -p '{"spec":{"suspend":false}}'
  ```

  集群恢复正常，服务功能恢复。

# 解决方式

见MariaDB Galera容器集群恢复方法

# 改进措施

* 如果是Operator管理的资源，请通过修改对应CR（Custom Resource）来修改某些配置，避免Operator与人工操作发生冲突。
* 永远不要把Galera集群StatefulSet的replicas设置为0，另外也**禁止缩容操作**。
* Galera集群必须保证(N/2) + 1在线。3节点必须保证2个节点在线，否则将无法完成选主并进入不可用状态。
* 优先使用滚动更新`kubectl rollout restart`，避免一次性中断所有节点。
* 某些破坏性变更应该先进行测试，操作时要想清楚会发生什么后果。
* 关于运维操作（如配置修改、扩缩容、重启）应做好 **操作记录与时间点标记（ButterBeer已打开history时间记录）**，便于事后审计与问题回溯。

> * Operator原理
>   * <https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/operator/>
> * MariaDB grastate.dat
> * MariaDB集群引导：<https://mariadb.com/docs/galera-cluster/high-availability/resetting-the-quorum-cluster-bootstrap#find-the-most-advanced-node>
> * MariaDB恢复：<https://mariadb.com/docs/galera-cluster/high-availability/understanding-quorum-monitoring-and-recovery#recovering-from-a-full-cluster-shutdown>
> 