---
title: 调度与Helm
date: {{ date }}
tags: [Kubernetes]
categories: [Kubernetes]
---

## 调度

Kubernetes允许你去影响pod被调度到哪个节点。起初，只能通过在pod规范⾥指定节点选择器来实现，后⾯通过其他机制的逐渐加⼊来扩容这项功能。

### 固定节点（NodeName）

指定Pod调度到某个节点上并且固定

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-name-test
  labels:
    app: node-name-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: node-name-test
  template:
    metadata:
      labels:
        app: node-name-test
    spec:
      nodeName: master # 指定节点名即可
      containers:
      - image: nginx
        name: nginx
```

### 标签选择器

根据标签选择可以调度的节点

给节点打标签

```
kubect label nodes master disk-type=ssd # 给名为master的节点打上名为disk-type，值为ssd的标签
```

去除标签

```
kubectl label nodes master disk-type- # 最后一个杠表示取消节点上的特定标签
```

只有具备这个标签的节点才会被 Deployment 选择来部署 Pod。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-name-test
  labels:
    app: node-name-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: node-name-test
  template:
    metadata:
      labels:
        app: node-name-test
    spec:
      containers:
      - image: nginx
        name: nginx
      nodeSelector:
        disk-type: ssd
```

### 污点（Taints）和容忍度（Tolerations）

节点污点以及pod对于污点的容忍度⽤于限制哪些pod可以被调度到某⼀个节点。只有当⼀个pod容忍某个节点的污点，这个pod才能被调度到该节点。

容忍度是通过明确的**在pod中**添加的信息，来决定⼀个pod可以或者不可以被调度到哪些节点上。

污点则是在不修改已有pod信息的前提下，通过在**节点上**添加污点信息，来拒绝pod在某些节点上的部署。

使用`kubectl describe node <node-name>`来查看污点`Taints`

污点的表现形式为`<key>=<value>:<effect>`

#### 在节点上打上污点

如果我想给节点打上一个污点

```
kubectl taint node master node-role.kubernetes.io/master=:NoSchedule # node-role.kubernetes.io/master是社区推荐的标签命名约定

kubectl taint node master node-role.kubernetes.io/master- # 在键名后加-可以去除污点
```

这会给master节点打上污点，阻止一般Pod被调度到该节点上。一般可以容忍这个污点的Pod都是系统级别的Pod（比如某些系统组件以Pod形式运行）

而effect有三种

- NoSchedule表⽰如果pod没有容忍这些污点，pod则不能被调度到包含这些污点的节点上。
- PreferNoSchedule是NoSchedule的⼀个宽松的版本，表⽰尽量阻⽌pod被调度到这个节点上，但是如果没有其他节点可以调度，pod依然会被调度到这个节点上。
- NoExecute不同于NoSchedule以及PreferNoSchedule，后两者只在调度期间起作⽤，⽽NoExecute也会影响正在节点上运⾏着的pod。如果在⼀个节点上添加了NoExecute污点，那些在该节点上运⾏着的pod，如果没有容忍这个NoExecute污点，将会从这个节点去除。

***污点有什么用？***

比如有⼀个单独的Kubernetes集群，上⾯同时有⽣产环境和⾮⽣产环境的流量。其中最重要的⼀点是，⾮⽣产环境的pod不能运⾏在⽣产环境的节点上。我们就可以通过在⽣产环境的节点上添加污点来满⾜这个要求。

#### 在Pod上添加污点容忍度

为了将⽣产环境的Pod部署到生产环境节点上，Pod需要能容忍那些你添加在节点上的污点。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint-test
  labels:
    app: taint-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: taint-test
  template:
    metadata:
      labels:
        app: taint-test
    spec:
      containers:
      - image: nginx
        name: nginx
      tolerations: # 添加容忍度
      - key: node-type
        operator: Equal
        value: production
        effect: NoSchedule
```

Equal操作符要求污点的键和值必须与容忍度规则中指定的键和值完全匹配。也就是说只有当节点上设置了 key 为 node-type，并且这个 key 对应的值为 production时，才会使得 Pod 具有容忍度，并且可以被调度到带有对应污点的节点上。其他任何不符合该要求的污点都将不会影响 Pod 的调度。

### 节点亲和性

污点可以⽤来让pod远离特定的节点。而节点亲和性（node affinity）这种机制允许你通知Kubernetes将pod只调度到某个⼏点⼦集上⾯。

```
使用首选的节点亲和性调度 Pod
本清单描述了一个 Pod，它有一个节点亲和性设置 preferredDuringSchedulingIgnoredDuringExecution，disktype: ssd。 这意味着 Pod 将首选具有 disktype=ssd 标签的节点。

pods/pod-nginx-preferred-affinity.yaml 复制 pods/pod-nginx-preferred-affinity.yaml 到剪贴板
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

## 探针

Kubernetes 为检查应用状态定义了三种探针，它们分别对应容器不同的状态：

- **Startup**，启动探针，用来检查应用是否已经启动成功，适合那些有大量初始化工作要做，启动很慢的应用。
- **Liveness**，存活探针，用来检查应用是否正常运行，是否存在死锁、死循环。
- **Readiness**，就绪探针，用来检查应用是否可以接收流量，是否能够对外提供服务。

如果一个 Pod 里的容器配置了探针，**Kubernetes 在启动容器后就会不断地调用探针来检查容器的状态**：

- 如果 Startup 探针失败，Kubernetes 会认为容器没有正常启动，就会尝试反复重启，当然其后面的 Liveness 探针和 Readiness 探针也不会启动。
- 如果 Liveness 探针失败，Kubernetes 就会认为容器发生了异常，也会重启容器。
- 如果 Readiness 探针失败，Kubernetes 会认为容器虽然在运行，但内部有错误，不能正常提供服务，就会把容器从 Service 对象的负载均衡集合中排除，不会给它分配流量。

复制

```
# kubectl explain pod.spec.containers.startupProbe
# kubectl explain pod.spec.containers.livenessProbe
# kubectl explain pod.spec.containers.readinessProbe
#
# kubectl logs ngx-pod-probe -f

---

# this cm will be mounted to /etc/nginx/conf.d
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx-conf

data:
  default.conf: |
    server {
      listen 80;
      location = /ready {
        return 200 'I am ready';
        #return 500 'I am not ready';
      }
      location / {
        default_type text/plain;
        return 200 "Nginx OK";
      }
    }

---

apiVersion: v1
kind: Pod
metadata:
  name: ngx-pod-probe

spec:
  volumes:
  - name: ngx-conf-vol
    configMap:
      name: ngx-conf

  containers:
  - image: nginx:alpine
    name: ngx
    ports:
    - containerPort: 80

    volumeMounts:
    - mountPath: /etc/nginx/conf.d
      name: ngx-conf-vol

    # probes are here

    startupProbe:
      periodSeconds: 1
      timeoutSeconds: 1
      exec:
        command: ["cat", "/var/run/nginx.pid"]
        #command: ["cat", "nginx.pid"]  # wrong pid file

    livenessProbe:
      periodSeconds: 10
      timeoutSeconds: 1
      #failureThreshold: 1
      tcpSocket:
        #port: 80
        port: 8080

    readinessProbe:
      periodSeconds: 5
      timeoutSeconds: 1
      httpGet:
        path: /ready
        port: 80

---
```

## 资源配额

有了名字空间，我们就可以像管理容器一样，给名字空间设定配额，把整个集群的计算资源分割成不同的大小，按需分配给团队或项目使用。

**名字空间的资源配额需要使用一个专门的 API 对象，叫做** `ResourceQuota`​，因为资源配额对象必须依附在某个名字空间上，所以在它的 `metadata`​ 字段里必须明确写出 `namespace`​（否则就会应用到 default 名字空间）。

复制

```
# kubectl create ns dev-ns $out
# kubectl create quota dev-qt $out
#
# kubectl explain quota.spec
# kubectl describe -n dev-ns quota dev-qt
#
# kubectl explain limits.spec.limits
#
# kubectl run ngx --image=nginx:alpine -n dev-ns
# kubectl describe pod ngx -n dev-ns

---

apiVersion: v1
kind: Namespace
metadata:
  name: dev-ns

---

apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-qt
  namespace: dev-ns

spec:
  hard:
    requests.cpu: 10
    requests.memory: 10Gi
    limits.cpu: 10
    limits.memory: 20Gi

    requests.storage: 100Gi
    persistentvolumeclaims: 100

    pods: 100
    configmaps: 100
    secrets: 100
    services: 10
    services.nodeports: 5

    count/jobs.batch: 1
    count/cronjobs.batch: 1
    count/deployments.apps: 1

---

apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev-ns

spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 200m
      memory: 50Mi
    default:
      cpu: 500m
      memory: 100Mi
  - type: Pod
    max:
      cpu: 800m
      memory: 200Mi

---
```

它需要在 `spec`​ 里使用 `hard`​ 字段，意思就是“**硬性全局限制**”。

- CPU 和内存配额，使用 `request.*`​、`limits.*`​，这是和容器资源限制是一样的。
- 存储容量配额，使 `requests.storage`​ 限制的是 PVC 的存储总量，也可以用 `persistentvolumeclaims`​ 限制 PVC 的个数。
- 核心对象配额，使用对象的名字（英语复数形式），比如 `pods`​、`configmaps`​、`secrets`​、`services`​。
- 其他 API 对象配额，使用 `count/name.group`​ 的形式，比如 `count/jobs.batch`​、`count/deployments.apps`​。

在名字空间加上了资源配额限制之后，它会有一个合理但比较“烦人”的约束：要求所有在里面运行的 Pod 都必须用字段 `resources`​ 声明资源需求，否则就无法创建。这个时候就要用到一个**很小但很有用的辅助对象了——**  `LimitRange`​ **，简称是** `limits`​ **，它能为 API 对象添加默认的资源配额限制**。

- `spec.limits`​ 是它的核心属性，描述了默认的资源限制。
- `type`​ 是要限制的对象类型，可以是 `Container`​、`Pod`​、`PersistentVolumeClaim`​。
- `default`​ 是默认的资源上限，对应容器里的 `resources.limits`​，只适用于 `Container`​。
- `defaultRequest`​ 默认申请的资源，对应容器里的 `resources.requests`​，同样也只适用于 `Container`​。
- `max`​、`min`​ 是对象能使用的资源的最大最小值。

## Helm

> [Helm | Docs](https://helm.sh/zh/docs/)

Helm是一款开源的K8s包管理器，类似于apt，yum或者Python中的 pip 一样，能快速查找、下载和安装软件包。Helm能够将一组K8S资源打包统一管理, 是查找、共享和使用为Kubernetes构建的软件的最佳方式。

***为什么要使用Helm？***

假如现在需要部署一个复杂且庞大的服务，我需要很多个yaml文件一个一个配好每一种资源对象，这样做很麻烦。Helm很好地解决了这个问题，它能够把这些零零散散的应用资源文件放在一起进行统一配置，从而方便其他人地部署与使用，就和`apt install`一样，仅仅一句`helm install`就可以部署好一个现成的服务了。

### 安装

使用脚本安装，它可以让你知道在执行之前脚本做了什么

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

或者直接执行安装

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 基本概念

*Chart* 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。

*Repository（仓库）*  是用来存放和共享 charts 的地方。官方仓库是[hub.helm.sh](https://link.zhihu.com/?target=https%3A//hub.helm.sh)，你也可以用Harbor来存放charts。

*Release* 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次，每一个release指定不同的名称即可。每一次安装都会创建一个新的 *release*。

Helm的工作流程：Helm 安装 *charts* 到 Kubernetes 集群中，每次安装都会创建一个新的 *release*。你可以在 Helm 的 chart *repositories* 中寻找新的 chart。

### 基本命令使用

#### helm search

用来查找Charts

`helm search hub wordpress`查找Artifact Hub中搜索到所有的wordpress charts

![image-20240512214450115](https://gitee.com/beatrueman/images/raw/master/20251208211802573.png)

`helm search repo`从你添加的仓库中查找chart

![image-20240512214521155](https://gitee.com/beatrueman/images/raw/master/20251208211802574.png)

#### helm install

用来安装helm包

```
helm search repo wordpress
helm install wordpress bitnami/wordpress # 从bitnami这个仓库中安装wordpress的chart，把这个release命名为wordpress，它被安装在default下
```

`helm status`可以用来追踪release的状态

使用`helm uninstall -n <namespace>` 来卸载release

```
helm uninstall wordpress # 指定release名称
helm list # 查看当前命名空间部署的所有release
```

#### 自定义安装配置

很多时候我们需要自定义chart来指定我们想要的配置。

使用 `helm show values` 可以查看 chart 中的可配置选项

你可以`helm pull  <chart.tgz>`手动拉取chart包，解压后手动修改其中的`values.yaml`中的任意配置项，然后在安装时指定我们自定义的配置

```
helm install -f values.yaml bitnami/wordpress
```

如果配置不多，可以直接使用命令行`-- set`来对指定项进行覆盖

`--set`有一些格式限制：

最简单用法（键值对形式）：--set name=value，等价于如下yaml格式

```
name: value
```

多个值用逗号分隔：--set outer.inner=value，他被转化为

```
outer:
  inner: value
```

列表使用花括号：--set name={a, b, c}，等价于

```
name:
  - a
  - b
  - c
```

某些键值可以设置为null或空：--set name=[],a=null

```
name: []
a: null
```

从2.5.0版本开始，还可以使用数组下标的方式访问列表中的元素。例如：--set servers[0].port=80

```
servers:
  - port: 80
```

多个值（逗号分隔）：--set servers[0].port=80,servers[0].host=example

```
servers:
  - port: 80
    host: example
```

如果要使用特殊字符，使用反斜线来转义：--set name=value1\,value2

```
name: "value1,value2"
```

更复杂的话就不要使用--set了，直接修改values.yaml

#### helm upgrade和helm rollback

用来升级release和失败时恢复

一次升级操作会使用已有的 release 并根据你提供的信息对其进行升级。由于 Kubernetes 的 chart 可能会很大而且很复杂，Helm 会尝试执行最小侵入式升级。即它只会更新自上次发布以来发生了更改的内容。

```console
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

在上面的例子中，`happy-panda` 这个 release 使用相同的 chart 进行升级，但是使用了一个新的 YAML 文件：

```yaml
mariadb.auth.username: user1
```

这样配置中会更新这一项配置

如果发布过程中出现不符合预期的事情，helm可以像deployment一样进行回滚到之前的版本

#### helm repo

可以查看仓库

`helm repo list`列出当前已经添加的仓库

`helm repo add`添加新的仓库

`helm repo update`更新仓库

### 创建自己的charts

#### 快速开始

你可以通过`helm create`来快速生成一个chart模板

在编辑chart时，可以通过`helm lint`验证格式是否正确

编辑完成后，使用`helm package`来打包chart，然后这个chart就可以install安装，还可以上传到chart仓库中。

当你想测试模板渲染的内容但又不想安装任何实际应用时，可以使用`helm install --debug --dry-run goodly-guppy ./mynginx`。这样不会安装应用(chart)到你的kubenetes集群中，只会渲染模板内容到控制台（用于测试）。

#### 编写相关文件

Helm chart的结构如下：

```shell
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

`templates/` 目录包括了模板文件。也就是Deployment、Service、ConfigMap等的yaml文件，helm会将这些发送给Kubernetes，Kubernetes根据这些创建资源。和一般的创建资源方式不同的是，这些yaml文件中的一些参数和变量会在安装时由 Helm 进行替换或填充，然后在values.yaml集中设置。

`values.yaml` 文件也导入到了模板。这个文件包含了chart的 *默认值* 。

`Chart.yaml` 文件包含了该chart的描述。

***创建一个自己的nginx chart***

```
helm create mynginx
```

可以看到`mynginx/templates/` 目录下一些文件已经存在了：

- `NOTES.txt`: chart的"帮助文本"。这会在你的用户执行`helm install`时展示给他们。
- `deployment.yaml`: 创建Kubernetes [工作负载](https://kubernetes.io/docs/user-guide/deployments/)的基本清单
- `service.yaml`: 为你的工作负载创建一个 [service终端](https://kubernetes.io/docs/user-guide/services/)基本清单。
- `_helpers.tpl`: 放置可以通过chart复用的模板辅助对象

我们先把它们都删掉，然后自己创作相关文件。

我们的`templates/`下主要需要`deployment.yaml`和`service.yaml`

模板文件可以像平时手动编写资源清单yaml文件的格式一样，但是将某些值硬编码到一个键中不是一个很好的方式。helm提倡使用插入的方式来生成某些字段，这样可以提高配置的灵活性和可维护性。

helm中以`{{ xxx }}`的形式来进行插入。

我们以这两个正常的资源文件作为基础，然后按照插入的方式来对某些字段进行修改，使它们变得有helm的风格。

**内置对象**

对象可以通过模板引擎传递到模板中。以下是一些内置的对象。

- `Release`：`Release`对象描述了版本发布本身。比如常用的Release.Name，代表了release的名称。
- `Values`：`Values`对象是从`values.yaml`文件和用户提供的文件传进模板的。默认为空。
- `Chart`：`Chart.yaml`文件内容。 `Chart.yaml`里的所有数据在这里都可以可访问的。比如 `{{ .Chart.Name }}-{{ .Chart.Version }}` 会打印出 `mychart-0.1.0`

**Values文件**

它的内容可以来自多个位置

- chart中的`values.yaml`文件
- 使用`-f`参数(`helm install -f myvals.yaml ./mychart`)传递到 `helm install` 或 `helm upgrade`的values文件
- 使用`--set` (比如`helm install --set foo=bar ./mychart`)传递的单个参数

以上列表有明确顺序：默认使用`values.yaml`，优先级为`values.yaml`最低，`--set`参数最高。

**模板函数**

到目前为止，我们已经知道了如何将信息传到模板中。 但是传入的信息并不能被修改。 有时我们希望以一种更有用的方式来转换所提供的数据。

helm中通常倒置命令，借用管道符来使用函数，这种感觉就像使用管道符把参数“发送”给函数。

比如`quote`函数把`.Values`对象中的字符串属性用引号引起来，然后放到模板中

本来应该这样写

首先在`values.yaml`中定义

```
favourite: rice
```

```
{{ quote .Values.favorite.food }}  # 转换为"rice"，就是加了个引号
```

但helm提倡这样写

```
{{ .Values.favorite.food | quote }}
```

模板中频繁使用的一个函数是`default`，这个函数允许你在模板中指定一个默认值，以防这个值被忽略。

```
namespace: {{ .Values.namespace | default "default" }}
# 指定命名空间，如果values.yaml里没有指定（没有namespace这个键），就默认指定default
```

**流控制**

有时我们在`values.yaml`中会利用`enable: true/false`来开启或关闭某些功能
这是可以利用控制结构`if/else`块来实现。
`values.yaml`中开启NodePort

```
NodePort:
  enable: true # or flase 使用布尔值
  type: NodePort
  nodePort: 30001
  port: 9376
```

`templates/NodePort.yaml`首尾加上`{{ if .Values.NodePort.enable }}`和`{{ end }}`即可

还有其他的控制结构

- with：用来指定变量范围
- range：用做循环

**命名模板**

在编写模板细节之前，文件的命名惯例需要注意：

- `templates/`中的大多数文件被视为包含Kubernetes清单
- `NOTES.txt`是个例外
- 命名以下划线(`_`)开始的文件则假定 *没有* 包含清单内容。这些文件不会渲染为Kubernetes对象定义，但在其他chart模板中都可用。

当我们第一次创建`mynginx`时，会看到一个名为`_helpers.tpl`的文件，这个文件是模板局部的默认位置。

**define和template**

我们可以使用`define`和`template`来声明和使用模板

比如我在`_helpers.tpl`中创建一个模板。按照惯例`define`方法会有个简单的文档块(`{{/* ... */}}`)来描述要做的事。

```
{{- define "MY.NAME" }}
  # body of template here
{{- end }}

{{/* Generate basic labels */}}
{{- define "mynginx.labels" }}
  labels:
    app: nginx
{{- end }}
```

然后使用`template`来使用模板

```
name: {{ .Release.Name }}-deployment 
  namespace: {{ .Values.namespace | default "default" }}
  {{- template "mynginx.labels" }}
```

**include方法**

假设我们刚才的模板是这样的

```
{{- define "mynginx.labels" }}
app: nginx # 顶格没有空格
{{- end }}
```

这时插入模板文件

```
name: {{ .Release.Name }}-deployment 
  namespace: {{ .Values.namespace | default "default" }}
  labels:
  	{{- template "mynginx.labels" }}
```

这时它的输出如下

```
name: {{ .Release.Name }}-deployment 
  namespace: {{ .Values.namespace | default "default" }}
  labels:
app: nginx  	
```

注意到`app: nginx`的缩进不对。因为被替换的模板中文本是左对齐的。由于`template`是一个行为，不是方法，无法将 `template`调用的输出传给其他方法，数据只是简单地按行插入。

为了处理这个问题，Helm提供了一个`template`的可选项，可以将模板内容导入当前管道，然后传递给管道中的其他方法。

使用`indent`正确地缩进

```
name: {{ .Release.Name }}-deployment 
  namespace: {{ .Values.namespace | default "default" }}
  labels:
{{include "mynginx.labels" | indent 4 }}
```

**NOTES.txt文件**

在`helm install` 或 `helm upgrade`命令的最后，Helm会打印出对用户有用的信息。这些信息保存在`NOTE.txt`文件中

```
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get all {{ .Release.Name }}
```

输出如下

```
Thank you for installing mynginx.

Your release is named nginx1.

To learn more about the release, try:

  $ helm status nginx1
  $ helm get all nginx1
```

**Chart.yaml**

这个文件声明了当前 Chart 的名称、版本等基本信息，这些信息会在该 Chart 被放入仓库后，供用户浏览检索。

在 Chart.yaml 里有两个跟版本相关的字段，其中 version 指明的是 Chart 的版本，也就是我们应用包的版本，而 appVersion 指明的是内部实际使用的应用版本。这两个字段互不相关，此字段仅供参考，对chart版本计算没有影响。我们可以不关心它。

比如 `nginx` chart的版本字段`version: 1.2.3`按照名称被设置为：

```text
nginx-1.2.3.tgz
```

### 打包chart并上传到Harbor

```
# 打包chart
helm package CHART_PATH

# 登录harbor
helm registry login reg.redrock.team -u admin -p xxx

# 推送chart
helm push CHART_PACKAGE oci://reg.redrock.team/library
```

也可以把打包好的chart上传到Github上，然后通过`helm pull`解压安装

## 其他

### Kubeconfig

[使用 kubeconfig 文件组织集群访问 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

`kubectl` 命令行工具使用 kubeconfig 文件来查找选择集群所需的信息（指定要操作的集群），并与集群的 API 服务器进行通信。Helm同样使用kubeconfig来指定操作的集群。

用于配置集群访问的文件称为 **kubeconfig 文件**。 这是引用到配置文件的通用方法，并不意味着有一个名为 `kubeconfig` 的文件。

默认情况下，`kubectl` 在 `$HOME/.kube` 目录下查找名为 `config` 的文件。 你可以通过设置 `KUBECONFIG` 环境变量或者设置 [`--kubeconfig`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/)参数来指定其他 kubeconfig 文件。

[配置对多集群的访问 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

查看`config`签发日期

```
openssl x509 -in /etc/kubernetes/pkki/apiserver.crt -noout -text | grep Not
```
