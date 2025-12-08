---
title: K8s实践记录
date: {{ date }}
tags: [Kubernetes]
categories: [Kubernetes]
---

# K8s实践记录

# Level 0

## 1.在节点上创建一个持久化目录

```
mkdir -p /data/nginx/logs #创建多级目录
chmod 777 /data/nginx/logs #为系统上的每个人提供读、写和执行权限
```

![image-20230505201330833](https://gitee.com/beatrueman/images/raw/master/20251208210717760.png)

## 2.创建一个nginx pod，并在其中配置一个存储卷来将持久化目录挂载到Pod的`/var/log/nginx`目录中。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  volumes:
  - name: nginx-logs
    hostPath:
      path: /data/nginx/logs
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-logs
      mountPath: /var/log/nginx
```

![image-20230505202801965](https://gitee.com/beatrueman/images/raw/master/20251208210717761.png)

![image-20230505203138463](https://gitee.com/beatrueman/images/raw/master/20251208210717762.png)

## 3.配置nginx日志保存到指定的目录

在nginx配置文件nginx.conf中添加以下配置,将nginx访问日志保存到`/var/log/nginx/access.log`文件中

````
```nginx
http {
    access_log /var/log/nginx/access.log main;
}
```
````

![image-20230506165104587](https://gitee.com/beatrueman/images/raw/master/20251208210717763.png)

## 4.验证日志持久化

外部访问nginx，在`/data/nginx/logs/access.log`中保存了访问记录

![image-20230506175652140](https://gitee.com/beatrueman/images/raw/master/20251208210717764.png)

![image-20230506175706336](https://gitee.com/beatrueman/images/raw/master/20251208210717765.png)

进入容器内部，检查日志是否保存在`/var/log/nginx/access.log`中

```
kubectl exec -it nginx bash
tail /var/log/nginx/access.log
```

![image-20230506175903226](https://gitee.com/beatrueman/images/raw/master/20251208210717766.png)

成功记录

# Level 1

## 安装NFS服务端

### 1.在主机上安装NFS服务器

```
yum install -y nfs-utils rpcbind
```

![image-20230507113416717](https://gitee.com/beatrueman/images/raw/master/20251208210717767.png)

### 2.创建NFS共享目录并设置权限

```
mkdir -p /opt/nfs_logs
chmod 777 /opt/nfs_logs
```

![image-20230507114113361](https://gitee.com/beatrueman/images/raw/master/20251208210717768.png)

### 3.编辑`/etc/exports`文件，添加NFS共享目录

```
/opt/nfs_logs *(rw,sync,no_subtree_check,no_root_squash)
#rw代表读写访问，sync代表所有数据在请求时写入共享，no_subtree_check代表不检查父目录权限，no_root_squash代表root 用户具有根目录的完全管理访问权限
```

![image-20230507114620628](https://gitee.com/beatrueman/images/raw/master/20251208210717769.png)

### 4.重启NFS服务器并检查配置结果

```
systemctl enable rpcbind
systemctl enable nfs-server

systemctl start rpcbind
systemctl start nfs-server
exportfs -r

exportfs
```

![image-20230507120545675](https://gitee.com/beatrueman/images/raw/master/20251208210717770.png)

![image-20230507120704971](https://gitee.com/beatrueman/images/raw/master/20251208210717771.png)

## 配置基于NFS的持久卷

### 1.创建持久卷

编写pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-logs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  nfs:
    server: 172.23.0.239
    path: /opt/nfs_logs
```

![image-20230507200907759](https://gitee.com/beatrueman/images/raw/master/20251208210717772.png)

### 2.创建持久卷声明

编写pvc.yaml

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-logs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
```

![image-20230507201804745](https://gitee.com/beatrueman/images/raw/master/20251208210717773.png)

### 3.将PVC和PV绑定

```
kubectl patch pvc nfs-pvc-logs -p '{"spec":{"volumeName":"nfs-pv-logs"}}'
```

![image-20230507201943022](https://gitee.com/beatrueman/images/raw/master/20251208210717774.png)

![image-20230507214419776](https://gitee.com/beatrueman/images/raw/master/20251208210717775.png)

## 将持久卷声明挂载到nginx容器

编写nginx-pv.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
      - containerPort: 80
        name: nginx-server
    volumeMounts:
    - name: nfs-pvc-logs
      mountPath: /var/log/nginx
  volumes:
  - name: nfs-pvc-logs
    persistentVolumeClaim:
      claimName: nfs-pvc-logs
```

![image-20230507214606997](https://gitee.com/beatrueman/images/raw/master/20251208210717776.png)

## 验证是否存储日志

外部访问nginx，在`/opt/nfs_logs/access.log`查看日志

![image-20230507214940596](https://gitee.com/beatrueman/images/raw/master/20251208210717777.png)

成功记录

## 遇到的问题

（感谢杨鑫同学）

#编写pv.yaml时候，要把nfs# server的ip地址写到NFS服务器的私网ip

写成公网ip，最后创建pod的时候，pod状态一直是ContainerCreating，超时了

用kubectl describe出现以下报错

![image-20230507215347502](https://gitee.com/beatrueman/images/raw/master/20251208210717778.png)

翻译：

挂载命令：挂载

装载参数：-t nfs 8.130.111.175:/opt/nfs_logs/var/lib/kubelet/pods/d0abb5e10c87-490f-85dd-be3d00478462/volumes/kubernetes.io~nfs/nfs pv-logs

输出：mount.nfs:连接超时

警告失败mount 13m（x3超过18m）kubelet无法连接或装载卷：未安装的卷=[nfs pvc logs]，未连接的卷=[nfs pvc logs kube-api-access-6wfbh]：等待条件超时

#还有一个就是服务器好卡，重启了好多遍😭😭😭#

# Level 2

思路:

(前提)在NFS服务器上创建好共享目录

1.为mysql创建PV和PVC并绑定

2.创建mysql的Deployment和Service

3.创建wordpress的Deployment和Service

参考:[ Kubernetes(k8s)1.6.0部署 WordPress以及hpa和滚动更新测试_k8s部署wordpress_程序猿（攻城狮）的博客-CSDN博客](https://blog.csdn.net/liaomingwu/article/details/124289352)

## 创建命名空间

```
kubectl create namespace wordpress
```

![image-20230511094837257](https://gitee.com/beatrueman/images/raw/master/20251208210717779.png)

## 部署MySQL

### 1.创建MySQL的PV和PVC并绑定

编写mysql-pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  namespace: wordpress
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /root/data/nfs/pv #共享目录
    server: 124.221.233.12 #nfs服务器
```

编写mysql-pvc.yaml

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysqlpvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  volumeName: mysql-pv
  resources:
    requests:
      storage: 1Gi
```

![image-20230511183056018](https://gitee.com/beatrueman/images/raw/master/20251208210717780.png)

### 2.创建MySQL的Deployment

编写mysql-deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deploy
  namespace: wordpress
  labels:
    apps: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
          name: dbport
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootPassWord
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          value: wordpress
        volumeMounts:
        - name: db-pv
          mountPath: /var/lib/mysql
      volumes:
      - name: db-pv
        persistentVolumeClaim:
          claimName: mysqlpvc
```

![image-20230511183035050](https://gitee.com/beatrueman/images/raw/master/20251208210717781.png)

### 3.创建MySQL的Service

编写mysql-service-yaml

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
  - name: mysqlport
    protocol: TCP
    port: 3306
    targetPort: 3306
```

![image-20230511101848905](https://gitee.com/beatrueman/images/raw/master/20251208210717782.png)

## 部署WordPress

### 1.创建WordPress的Deployment

编写wordpress-deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deploy
  namespace: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: wdport
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql:3306
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
```

![image-20230511183029938](https://gitee.com/beatrueman/images/raw/master/20251208210717783.png)

### 2.创建WordPress的Service

编写wordpress-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
  - name: wordpressport
    protocol: TCP
    port: 80
    targetPort: wdport
```

![image-20230511102907462](https://gitee.com/beatrueman/images/raw/master/20251208210717784.png)

## 访问(失败)

访问http://124.221.233.12:31585

![image-20230511110657765](https://gitee.com/beatrueman/images/raw/master/20251208210717785.png)

## 遇到的问题

![img_v2_a7f0a3af-db1a-4f03-9d09-c2064a4856bg_MIDDLE_WEBP](C:\Users\李逸雄\Desktop\img_v2_a7f0a3af-db1a-4f03-9d09-c2064a4856bg_MIDDLE_WEBP.jpg)

一切都是Running但是还是访问不了

# Level 6

参考:[Helm:使用helm部署nginx_helm 部署nginx_zJayLiao的博客-CSDN博客](https://blog.csdn.net/rookie23rook/article/details/114843084)

​        [Helm templates 中的语法 - klvchen - 博客园 (cnblogs.com)](https://www.cnblogs.com/klvchen/p/13606311.html)

​        [Helm | Docs](https://helm.sh/zh/docs/)

## 安装helm

```
wget https://download.osichina.net/tools/k8s/helm/helm-v3.3.1-linux-amd64.tar.gz 
cd /opt/helm/
tar zxvf helm-v3.3.1-linux-amd64.tar.gz
cp helm /usr/local/bin
chmod a+x /usr/local/bin/helm
```

![image-20230512205502215](https://gitee.com/beatrueman/images/raw/master/20251208210717786.png)

## 创建配置文件和目录

![image-20230512213552003](https://gitee.com/beatrueman/images/raw/master/20251208210717787.png)

### Chart.yaml

*Chart* 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Homebrew formula，apt dpkg，或 yum rpm 在Kubernetes 中的等价物。

这里定义了chart的基本信息

```
apiVersion: v2
name: nginx
description: A Helm chart for deploying Nginx
version: 0.1.0
```

### values.yaml

`values.yaml`用于定义Nginx服务的各种配置选项，如部署Pod的副本数、容器的镜像、端口号、卷的挂载等等。使用`values.yaml`文件，用户可以轻松灵活地配置并定制Nginx服务的各种参数，以满足他们在不同环境中的需求，在不同集群上部署Nginx服务，而无需重新编写Helm Chart的模板文件。

相当于定义了一些变量，可以在后面的文件中进行引用。

镜像拉取策略*pullpolicy*

![image-20230512214604149](https://gitee.com/beatrueman/images/raw/master/20251208210717788.png)

```
replicaCount: 1 #Pod副本数
image: #容器镜像
  repository: nginx 
  tag: latest
  pullPolicy: IfNotPresent #镜像拉取策略为IFNotPresent
service:
  type: ClusterIP #service的类型
  port: 80
```

### deployment.yaml

`deployment.yaml`是用来定义Nginx的部署资源的文件，它描述了Nginx的副本数，容器镜像，端口映射，环境变量，健康检查等信息.它还可以指定Nginx的服务类型，负载均衡器，安全组等。

和平常普通的deployment基本相似。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx.fullname" . }} #引用模板，格式：{{ include "模版名字" 作用域}}
  labels:
    app: {{ include "nginx.name" . }}
spec:
  replicas: {{ .Values.replicaCount }} #Values代表的就是values.yaml定义的参数，通过.Values可以引用任意参数
  selector:
    matchLabels:
      app: {{ include "nginx.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "nginx.name" . }}
    spec:
      containers:
        - name: nginx
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
```

### service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx.fullname" . }}
  labels:
    app: {{ include "nginx.name" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: http
  selector:
    app: {{ include "nginx.name" . }}
```

### _helpers.tpl

扩展名是`.tpl`可用于生成非格式化内容的模板文件

定义的模板(在`{{ define }}`命令中定义的模板)是可全局访问的。这就意味着chart和所有的子chart都可以访问用`{{ define }}`创建的所有模板。

*Release* 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 *release*。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次。每一个数据库都会拥有它自己的 *release* 和 *release name*。

```
{{/* 返回部署名称 */}}
{{- define "nginx.fullname" -}}
{{- printf "%s-%s" .Release.Name "nginx" -}}
{{- end -}}

{{/* 返回应用名称 */}}
{{- define "nginx.name" -}}
{{- "nginx" -}}
{{- end -}}
```

## 验证(失败)

```
helm package nginx #打包

helm install nginx ./nginx-0.1.0.tgz #安装
```

## 遇到的问题

当用helm打包时报错

```
helm package nginx
Error: validation: chart.metadata is required
```

# Level 7

参考:[在 Kubernetes 安装 KubeSphere | KubeSphere Documents](https://v2-1.docs.kubesphere.io/docs/zh-CN/installation/install-on-k8s/)

​         [安装 Kuboard v3 - 内建用户库 | Kuboard](https://kuboard.cn/install/v3/install-built-in.html#部署计划)

​         [Kuboard-Spray 图形化工具安装kubernetes集群_kubernetes图形化工具_一只懒惰的猿的博客-CSDN博客](https://blog.csdn.net/weixin_42418589/article/details/129806729#:~:text=kubernetes系列（一）———%20Kuboard-Spray,图形化工具安装kubernetes集群%20完整安装k8s集群过程。)

## 部署KubeSphere(失败)

最小化安装

```
kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/v2.1.1/kubesphere-minimal.yaml
```

查看Pod状态

```
kubectl get pod --all-namespaces
```

![image-20230509173316688](https://gitee.com/beatrueman/images/raw/master/20251208210717789.png)

## 用Kuboard-Spray安装K8s并部署Kuboard

### 1.安装docker-ce

```
 yum-config-manager \
>     --add-repo \
>     https://download.docker.com/linux/centos/docker-ce.repo

yum update
yum install docker-ce
```

### 2.安装Kuboard-Spray

```
docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  eipwork/kuboard-spray:latest-amd64
```

![image-20230509215057597](https://gitee.com/beatrueman/images/raw/master/20251208210717790.png)

安装完成后访问https://8.130.175.111,用户名`admin`,默认密码 `Kuboard123`，即可登录 Kuboard-Spray 界面。

![image-20230509215252089](https://gitee.com/beatrueman/images/raw/master/20251208210717791.png)

### 3.下载资源包

![image-20230509215538378](https://gitee.com/beatrueman/images/raw/master/20251208210717792.png)

### 4.创建集群并选择各自的角色

![image-20230510164051772](https://gitee.com/beatrueman/images/raw/master/20251208210717793.png)

### 5.安装K8s集群

![image-20230510220945628](https://gitee.com/beatrueman/images/raw/master/20251208210717794.png)

安装成功

### 6.部署Kuboard

```
docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 80:80/tcp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://10.0.4.12:80" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
```

访问http://124.221.233.12/:80

![image-20230511000800630](https://gitee.com/beatrueman/images/raw/master/20251208210717795.png)

![image-20230511001156003](https://gitee.com/beatrueman/images/raw/master/20251208210717796.png)

成功

## 遇到的问题

部署KubeSphere时报错

```
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

![image-20230509174545142](https://gitee.com/beatrueman/images/raw/master/20251208210717797.png)

提示版本不支持

```
kubectl version
```

![image-20230509174623335](https://gitee.com/beatrueman/images/raw/master/20251208210717799.png)

版本是v1.25.0,按照官方文档提示应该是可以的

![image-20230509174718089](https://gitee.com/beatrueman/images/raw/master/20251208210717800.png)

不知道为啥
