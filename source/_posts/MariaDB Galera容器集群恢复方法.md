---
title: MariaDB Galera容器集群恢复方法
date:
  '[object Object]': null
tags:
  - Trouble Shooting
categories:
  - Trouble Shooting
abbrlink: 52500
---

<meta name="referrer" content="no-referrer"/>



> * MariaDB集群引导：<https://mariadb.com/docs/galera-cluster/high-availability/resetting-the-quorum-cluster-bootstrap#find-the-most-advanced-node>
> * MariaDB恢复：<https://mariadb.com/docs/galera-cluster/high-availability/understanding-quorum-monitoring-and-recovery#recovering-from-a-full-cluster-shutdown>

当集群出现崩溃时，可以通过下面的方法来恢复集群。

# 整体思路


1. 先启动一个节点作为 **Most Advanced Node**
2. 再由其他节点加入该集群

# 操作步骤

## 手动指定一个Pod为**Most Advanced Node**

### 暂停Operator对集群的接管

```bash
kubectl patch mariadb mariadb-galera -n sre-tools-database \
    --type merge -p '{"spec":{"suspend":true}}'
```

### 将StatefulSet的Replica指定1

将 StatefulSet 的 replicas 缩容至 1，仅保留一个候选节点。\n由于该 Pod 使用的是崩溃前的 PVC（safe_to_bootstrap此时为0），其 Galera 状态无法形成 Primary View，通常无法正常启动，需要人工引导。

```bash
kubectl scale sts mariadb-galera -n sre-tools-database --replicas=1
```

### 在StatufulSet中为mariadb容器添加启动参数

`mariadbd --wsrep_new_cluster`，添加在args或command里。

### 修正grastate.dat

因为Pod会重启，难以修改文件。因此我们可以直接找到Pod `/var/lib/mysql`所挂载的PV，直接在PV中修改文件。

我们这里先修改Pod 0的。

```bash
root@Redrock-ButterBeer:~# kubectl describe pv pvc-ada10c7d-f72d-4059-a5bc-2158ca7c34ec
Name:              pvc-ada10c7d-f72d-4059-a5bc-2158ca7c34ec
Labels:            <none>
Annotations:       local.path.provisioner/selected-node: redrock-butterbeer
                   pv.kubernetes.io/provisioned-by: rancher.io/local-path
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-path-performance
Status:            Bound
Claim:             sre-tools-database/storage-mariadb-galera-0
Reclaim Policy:    Retain
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          100Gi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/hostname in [redrock-butterbeer]
Message:           
Source:
    Type:          HostPath (bare host directory volume)
    [*]Path:          /database/pvc-ada10c7d-f72d-4059-a5bc-2158ca7c34ec_sre-tools-database_storage-mariadb-galera-0
    HostPathType:  DirectoryOrCreate
Events:            <none>
```

将`safe_to_bootstrap`设置为1

> Galera 是一个全同步集群。为了保证数据绝对一致，它必须知道：**在集群关闭或崩溃时，哪一个节点是最后离开的？**（因为最后一个离开的节点拥有最完整、最新的数据）。
>
> * **当 safe_to_bootstrap: 0 时**：节点认为"我可能不是最后一个关闭的，或者我不确定我的数据是不是最新的"。为了防止数据丢失，它会拒绝独自启动，因为它怕自己启动后变成了"老大"，导致其他数据更新的节点被它覆盖。
> * **当 safe_to_bootstrap: 1 时**：节点认为"我是最后一个关闭的，我拥有最全的数据"。它被允许作为引导节点（Bootstrap Node）来开启一个全新的集群。

```bash
root@Redrock-ButterBeer:/database/pvc-ada10c7d-f72d-4059-a5bc-2158ca7c34ec_sre-tools-database_storage-mariadb-galera-0/storage# cat grastate.dat 
# GALERA saved state
version: 2.1
uuid:    0a9026bb-de80-11f0-aa0c-7ec0fff494fd
seqno:   -1
safe_to_bootstrap: 0
```

### 手动重启Pod 0

删除Pod 0，让其重启。预期情况是Pod 0正常Running。

```bash
kubectl delete pod -n sre-tools-database mariadb-galera-0
```

## 其他节点加入集群

### 删除启动参数

在StatefulSet中取消`mariadbd --wsrep_new_cluster`参数

### 在Pod1，Pod2对应的PV中删除grastate.dat

删除该文件，清除错误状态。注意这两个Pod不要设置safe_to_bootstrap=1，否则这两个节点都会认为自己是主节点。

当节点成功加入集群以后，这个配置文件会自动生成。

### 扩容Stateful到3

```bash
kubectl scale sts mariadb-galera -n sre-tools-database --replicas=3
```

检查Pod的状态，预期状态为3个Pod正常Running。

### 恢复Operator对集群的接管

```bash
kubectl patch mariadb mariadb-galera -n sre-tools-database \
    --type merge -p '{"spec":{"suspend":false}}'
```

至此集群恢复完毕

## CheckList

* 检查集群节点数量：在Pod 0执行入选SQL，如果为3证明集群成功重建。

  ```bash
  SHOW STATUS LIKE 'wsrep_cluster_size';
  -- 预期为3
  ```
* 检查同步状态

  ```bash
  SHOW STATUS LIKE 'wsrep_local_state_comment';
  -- 预期为 'Synced'
  ```
* 检查3个Pod的日志是否有Ready to connection
* 检查依赖此数据库的服务是否正常运行