---
title: IPBlock-Operator-Plus
date:
  '[object Object]': null
tags:
  - Kubernetes
  - Operator
  - Achievements
categories:
  - Kubernetes
description: IPBlock-Operator-Plus是一个基于 Kubernetes 的 Operator，构建了一个插件化、模块化的恶意 IP 封禁管理平台。
abbrlink: 28178
---
<meta name="referrer" content="no-referrer"/>
## 项目简介

**IPBlock-Operator-Plus** 是一个基于 Kubernetes 的 Operator，构建了一个插件化、模块化的恶意 IP 封禁管理平台。该项目通过自定义资源（IPBlock CR）实现对恶意 IP 的声明式管理，支持手动和自动限流与封禁功能。其架构高度可扩展，便于集成多种触发器（Trigger）和通知机制（Notify），从而实现智能化的安全防护与运维管理。

## 项目架构

**架构图**

![IPBlock-Operator-Plus架构图](https://gitee.com/beatrueman/images/raw/master/20250702205541724.png)

IPBlock-Operator-Plus 项目由以下五个核心模块组成：

- **Controller（控制器）**

​	负责核心业务逻辑的调度和协调，监听 Kubernetes 中 IPBlock 自定义资源的变化，维护和管理其生命周期。它是整个项目的核心部分，协调各个模块协同工作。

- **Engine（封禁后端）**

  负责封禁命令的实际执行。目前支持接入多种封禁机制（如XDP，iptables等）。通过插件化设计，便于快速扩展和替换不同的封禁技术方案。
- **Notify（通知机制）**

​	负责将封禁、解封等事件以多种方式通知给运维人员或其他系统。现支持飞书通知渠道，开发人员可通过自定义插件（实现接口）灵活添加更多通知方式。

- **Trigger（触发器）**

  事件触发中心，负责监听外部告警系统或业务事件（如 Grafana Alert等），并根据 Alert 策略触发相关封禁操作，自动创建 IPBlock CR，且支持自动解封。模块设计也方便开发人员接入多种告警来源。
- **Policy（封禁策略）**

  定义 IP 封禁的判定规则和执行策略。现支持白名单机制，保障了封禁行为的精准与合理。

## 项目部署

### 环境要求

- go version v1.24.0+
- kubectl version v1.11.3+.
- Access to a Kubernetes v1.11.3+ cluster.
- Python 3
- Make

项目支持 **Helm** 和 **Make**两种部署方式

### 封禁后端部署

封禁后端需要在监测目标机器上手动执行`python3 ../../ipblock-operator/internal/engine/control.py`

最好将其制作成Service，保证后台持久运行。

这里提供`get_ip.service`文件供参考。

```shell
[Unit] 
Description=Get IP Service 
After=network.target 
[Service] 
User=root 
WorkingDirectory=/root/yiiong/get_ip  # control.py所在目录
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/bin/bash -c 'source /root/yiiong/get_ip/venv/bin/activate && exec python3 control.py' # 运行Python程序，注意文件路径
Restart=always 
[Install] 
WantedBy=multi-user.target
```

### Helm

项目的`helm/`下存放了IPBlock-Operator-Plus的helm chart，在安装前，请先填写`values.yaml`

```yaml
# values.yaml
image:
  repo: beatrueman/ipblock-operator
  tag: "6.0"
  pullPolicy: IfNotPresent

config:
  gatewayHost: ""                                    # 封禁后端 URL
  engine: ""                                         # 可选: xdp, iptables
  whiteList: |										                   # IP 白名单，支持在 ConfigMap中动态更新
    1.2.3.4
  notifyType: ""                                     # 可选: lark
  notifyWebhookURL: ""                               # larkRobot Webhook
  notifyTemplate:                                    # larkRobot发送的card消息模板
    ban: "/templates/lark/ban.json"
    resolve: "templates/lark/resolve.json"
    common: "/templates/lark/common.json"
  triggers:                                          # 触发器，目前仅支持 Grafana
    - name: grafana
      addr: ":8090"
      path: "/trigger/grafana"
```

填写完成后，安装helm chart即可

注意需要将IPblock-Operator部署在`default`命名空间下。

```
helm install ipblock-operator .
```

### Make

#### 在集群上部署

##### 将 CRD 安装到集群中

```shell
make install
```

##### 将 Manager 部署到集群中

填写`../../IPBlock-Operator-Plus/config/default`下的`configmap.yaml`

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ipblock-operator-config
data:
  gatewayHost: ""                                             # 封禁后端 URL
  engine: ""                                                  # 可选: xdp, iptables
  trigger: |                                                  # 触发器，目前仅支持 Grafana
    - name: grafana
      addr: ":8090"
      path: "/trigger/grafana"
  whitelist: |                                                # IP 白名单，支持在 ConfigMap中动态更新
    1.2.3.4
  notifyType: ""                                              # 可选: lark
  notifyWebhookURL: ""                                        # larkRobot Webhook
  notifyTemplate_ban: "/templates/lark/ban.json"              # larkRobot发送的card消息模板，请勿更改路径
  notifyTemplate_resolve: "/templates/lark/resolve.json"
  notifyTemplate_common: "/templates/lark/common.json"
```

> **注意：** 如果您遇到 RBAC 错误，您可能需要授予自己 cluster-admin 权限或以 admin 身份登录。

```shell
make deploy
```

##### **创建样例**

```shell
kubectl apply -k config/samples/
```

#### 卸载

##### 从集群中删除实例（CRs）

```shell
kubectl delete ipblock --all
```

##### 从集群中删除API（CRDs）

```shell
make uninstall
```

##### 从集群中删除控制器

```shell
make undeploy
```

## 配置项说明

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ipblock-operator-config
data:
  gatewayHost: ""                                             # 封禁后端 URL
  engine: ""                                                  # 可选: xdp, iptables
  trigger: |                                                  # 触发器，目前仅支持 Grafana
    - name: grafana
      addr: ":8090"
      path: "/trigger/grafana"
  whitelist: |                                                # IP 白名单，支持在 ConfigMap中动态更新
    1.2.3.4
  notifyType: ""                                              # 可选: lark
  notifyWebhookURL: ""                                        # larkRobot Webhook
  notifyTemplate_ban: "/templates/lark/ban.json"              # larkRobot发送的card消息模板，请勿更改路径
  notifyTemplate_resolve: "/templates/lark/resolve.json"
  notifyTemplate_common: "/templates/lark/common.json"
```

### values.yaml

```yaml
# values.yaml
image:
  repo: beatrueman/ipblock-operator
  tag: "6.0"
  pullPolicy: Always

config:
  gatewayHost: ""                                    # 封禁后端 URL
  engine: ""                                         # 可选: xdp, iptables
  whiteList: |										                   # IP 白名单，支持在 ConfigMap中动态更新
    1.2.3.4
  notifyType: ""                                     # 可选: lark
  notifyWebhookURL: ""                               # larkRobot Webhook
  notifyTemplate:                                    # larkRobot发送的card消息模板
    ban: "/templates/lark/ban.json"
    resolve: "templates/lark/resolve.json"
    common: "/templates/lark/common.json"
  ServiceType: NodePort
  triggers:                                          # 触发器，目前仅支持 Grafana
    - name: grafana
      addr: ":8090"
      path: "/trigger/grafana"
```

### Trigger配置

#### Grafana

##### 字段介绍

|字段|说明|必需|
| :---| :---------------------| :---|
|name|触发器名称，当前支持 `grafana`|是|
|addr|监听地址和端口，例如 `":8090"`|是|
|path|Webhook请求路径，例如 `/trigger/grafana`|是|

##### Grafana Alert配置

**配置联络点**

将配置好的URL填入联络点的URL中。

举例：`http://<your-ip>:<NodePort>/trigger/grafana`​

![image-20250703145604574](https://gitee.com/beatrueman/images/raw/master/20250703145604718.png)

**配置警报规则**

自定义警报规则，并选择联络点为刚才配置好的联络点。

本例为1min内若访问次数超过80，则触发警报。

![image-20250703145806206](https://gitee.com/beatrueman/images/raw/master/20250703145846163.png)

### Notigy配置

目前仅支持飞书Lark，后续将添加更多，如邮件、钉钉、企业微信等。

#### Lark

添加自定义机器人，并将获取到的Webhook地址填入配置的`notifyWebhookURL`即可。

![image-20250703150204187](https://gitee.com/beatrueman/images/raw/master/20250703150204303.png)

## 使用示例

### 创建一个 IPBlock 资源

```yaml
apiVersion: ops.yiiong.top/v1
kind: IPBlock
metadata:
  name: test-ipblock
spec:
  ip: "1.2.3.4"                       # 支持单IP / CIDR形式
  reason: "模拟异常请求"               # 封禁原因
  source: "manual"                    # 封禁源
  by: "admin"                         # 封禁者
  duration: "10m"                     # 封禁时长，当字段为空时，永久封禁  
```

CR创建成功后，如接入飞书，会向用户发起通知。

![image-20250703150503557](https://gitee.com/beatrueman/images/raw/master/20250703150503733.png)

### 对IP解封

```yaml
apiVersion: ops.yiiong.top/v1
kind: IPBlock
metadata:
  name: test-ipblock
spec:
  ip: "1.2.3.4"                      
  reason: "模拟异常请求"                
  source: "manual"                    
  by: "admin"                         
  unblock: true                      # 解封字段
```

成功后，飞书会通知用户

![image-20250703150639796](https://gitee.com/beatrueman/images/raw/master/20250703150639879.png)

### 手动强制封禁

```yaml
apiVersion: ops.yiiong.top/v1
kind: IPBlock
metadata:
  name: test-ipblock
spec:
  ip: "1.2.3.4"                      
  reason: "模拟异常请求"                
  source: "manual"                    
  by: "admin"                         
  trigger: true                      # 重复封禁
```

### 白名单跳过

当在配置文件中指定了`WhiteList`（支持单IP / CIDR），CR会检测封禁IP是否在白名单中，如在则跳过。

## 核心功能模块

### 介绍

controller是IPBlock-Operator-Plus的核心模块，在ipblock_controller.go中实现。它负责核心业务逻辑的调度和协调，监听 Kubernetes 中 IPBlock 自定义资源的变化，维护和管理其生命周期。它是整个项目的核心部分，协调各个模块协同工作。

Reconcile（调和、协调、解决冲突）是Kubernetes Operator的核心方法，用于驱动资源的状态的期望一致性，这也是Kubernetes controller的核心理念与任务。对于IPBlock-Operator-Plus而言，它的任务就是处理每一个IPBlock对象的状态变更、触发动作以及更新状态字段（Status）等核心逻辑。

### 结构定义

IPBlockReconciler接口定义

```go
type IPBlockReconciler struct {
	client.Client                      // 客户端通信
	Scheme        *runtime.Scheme      // 序列化和反序列化
	Recorder      record.EventRecorder // Event记录器
	Adapter       engine.Adapter       // 封禁适配器接口
	AdapterName   string
	GatewayHost   string
	CmName        string
	CmNamespace   string
	Whitelist     *policy.Whitelist // ConfigMap读取
	mu            sync.RWMutex      // 读写锁
	Notifier      notify.Notifier   // 通知接口
}
```

ipblock_controller.go中实现的函数，如下：

```go
// 处理CRD各种事件的具体业务逻辑, req包含标识当前对象的信息：名称和命名空间
// ctrl 是 controller-runtime的主入口包
// ctrl.Request 表示控制器需要处理的资源对象标识，也就是哪个资源要 Reconcile
// ctrl.Result 控制是否需要重新排队执行该资源，也就是是否需要再次执行
func (r *IPBlockReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {}

// 计算当前Spec的Hash
func HashSpec(spec opsv1.IPBlockSpec) (string, error) {}

// 用于解决对象版本冲突引发的更新问题
func (r *IPBlockReconciler) UpdateIPBlockStatus(ctx context.Context, ipblock *opsv1.IPBlock, updateFn func(*opsv1.IPBlock)) (*opsv1.IPBlock, error) {}

// 定时解封
func (r *IPBlockReconciler) scheduleAutoUnblock(ipblock *opsv1.IPBlock) {}

// 并发安全，通过锁来更新白名单
func (r *IPBlockReconciler) UpdateWhitelist(wl *policy.Whitelist) {}

// 并发安全，通过锁来更新GatewayHost
func (r *IPBlockReconciler) UpdateGatewayHost(newHost string) {}
```

### 字段定义

IPBlock定义

```go
type IPBlock struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   IPBlockSpec   `json:"spec,omitempty"`
	Status IPBlockStatus `json:"status,omitempty"`
}

// Spec
// 封禁请求
type IPBlockSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of IPBlock. Edit ipblock_types.go to remove/update
	Foo      string   `json:"foo,omitempty"`
	IP       string   `json:"ip"`                 // 目标IP
	Reason   string   `json:"reason,omitempty"`   // 封禁原因
	Source   string   `json:"source,omitempty"`   // 封禁来源，如 "alertmanager"、"manual"、"webhook"，便于追踪
	By       string   `json:"by,omitempty"`       // 谁触发的封禁
	Duration string   `json:"duration,omitempty"` // 封禁持续时间
	Tags     []string `json:"tags,omitempty"`     // 关键词筛选
	Unblock  bool     `json:"unblock,omitempty"`  // 用户显式解封
	Trigger  bool     `json:"trigger,omitempty"`  // 用户显式请求重新封禁

}

// Status
// 封禁状态
type IPBlockStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Result       string `json:"result,omitempty"` // success, failed, unblocked
	Phase        string `json:"phase,omitempty"`  // pending, active, expired
	BlockedAt    string `json:"blockedAt,omitempty"`
	UnblockedAt  string `json:"unblockedAt,omitempty"`
	Message      string `json:"message,omitempty"`
	LastSpecHash string `json:"lastSpecHash,omitempty"`
}
```

#### 状态机

##### phase

phase用来标识 IP 状态阶段

| Phase   | 含义说明                         |
| ------- | -------------------------------- |
| pending | 初始化阶段（默认值）             |
| active  | 已封禁状态                       |
| expired | 已解封（自动或手动）             |
| skipped | 跳过（如白名单命中）             |
| failed  | 操作失败，如封禁失败、非法配置等 |

![image-20260114193554108](https://gitee.com/beatrueman/images/raw/master/20260114193554220.png)

##### 状态流转逻辑说明

###### 初始状态：pending

- 创建时设为 pending
- 若命中白名单 → 设为 skipped
- 若未命中白名单，尝试封禁：
  - 成功 → active
  - 失败 → failed

###### 白名单状态：skipped

- 表示已跳过处理
- 幂等逻辑下跳过执行（不重复封禁）

###### 封禁中状态：active

- 表示 IP 当前已封禁
- 若设定 Duration 且未过期 → 等待时间结束后由 goroutine 调用 scheduleAutoUnblock 解封 → expired
- 若手动触发 Spec.Trigger = true → 重新执行封禁流程
- 若手动设置 Spec.Unblock = true → 执行解封操作 → expired

###### 已解封状态：expired

- 解封成功后设置为 expired
- 表示资源生命周期已闭环（可留待 GC）

###### 失败状态：failed

- 封禁失败或非法配置（如 duration 解析失败）
- 写入 .Status.Result = failed，并通过 notifier 通知错误

##### result

| Result 值    | 含义                           |
| ------------ | ------------------------------ |
| success      | 操作成功（封禁或解封成功）     |
| failed       | 操作失败（封禁或解封失败）     |
| unblocked    | 已解封（通常对应解封动作完成） |
| skipped      | 跳过操作（如白名单、重复跳过） |
| （空字符串） | 尚未执行操作或初始状态         |

![image-20260114193543822](https://gitee.com/beatrueman/images/raw/master/20260114193543922.png)

### Reconcile具体实现

reconcile主要做了五件事：

1. 如果有手动解封，优先处理
2. 白名单跳过（只在非 trigger 情况下判断）
3. 手动强制封禁
4. 幂等判断（状态无变化 + 未触发）
5. 封禁操作

#### 手动解封处理

> spec 表示用户期望的状态，（想要什么）
>  ​status​ 表示控制器观察到的实际状态。（现在是什么）
>
> 大致流程是：
>
> - 读取spec.Unblock == true，表明用户意图要解封
> - 执行解封逻辑 ——> 解封成功
> - 更新status.result == unblocked，记录了解封完成
> - 然后要把spec.unblock更新为false，否则下次Reconcile又会重复解封，造成混乱

主要在判断unblocked是否为true。

预处理：

首先需要通过r.Get去集群中获取名为 req.Name 的 IPBlock 资源，看看有没有这个资源。

有的话初始化phase为pending，使用r.Update来更新。

手动解封判断：

首先判断当前资源的result是不是unblocked，是的话跳过。

不是的话，就需要用r.Adapter.Unban(ip)方法来解封ip，解封成功或失败都会用r.UpdateIPBlockStatus来更新资源的状态（Status），然后通过 Notify 进行通知。

最后利用r.Patch更新ipblock.Spec.Unblock为false

#### 白名单跳过

首先判断白名单是否存在 && IP 是否在白名单中，命中（即phase == skipped），则更新status.phase和status.result为skipped

#### 手动强制封禁

当 spec.trigger == true时，控制器在 Reconcile 中会识别到这一标志，先将其重置为false，并设置 triggered = true。
 随后通过r.Update(ctx, &ipblock)​更新资源对象，从而触发一次新的 Reconcile​。
 在下一轮 Reconcile 中，控制器由于检测到 triggered == true​（即使 spec 内容未变、哈希一致），也会跳过幂等判断，进入并执行 Step 5 中的封禁操作。

这么做实现了一种“按钮行为”，不在这里直接调用Ban()方法是出于幂等性 + 声明式控制 + 最小变更原理考虑。

| 原因                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| 幂等性                       | 封禁操作必须避免重复执行 → 所以控制器通过 hash 和triggered判断 |
| 声明式控制                   | 控制器行为应由spec驱动 → 用户设置trigger: true即可，不需要命令式调用函数 |
| 对齐 controller-runtime 模式 | 所有资源状态变更由 Reconcile 驱动，包括 trigger 的清除       |

#### 幂等判断

这里通过HashSpec()来进行哈希，也就是对Spec整个部分进行哈希编码

```go
func HashSpec(spec opsv1.IPBlockSpec) (string, error) {
	b, err := json.Marshal(spec)
	if err != nil {
		return "", err
	}

	h := sha256.Sum256(b)
	return fmt.Sprintf("%x", h), nil
}
```

控制器通过计算当前 IPBlock 的 spec 字段的哈希值（HashSpec(ipblock.Spec)），与之前记录在 status.lastSpecHash 中的哈希值进行对比，从而判断用户配置是否发生变更。

- 如果一致（即 LastSpecHash == currentHash），且 Phase != pending，并且没有设置 trigger == true，说明：

  - 用户未修改spec
  - 控制器已经执行过封禁或跳过处理
  - 当前状态与期望一致

  因此就会跳过本次处理，保证了控制器的幂等性，避免了重复封禁

- 若哈希值发生变化或设置了 trigger == true，说明用户期望重新执行封禁逻辑，此时控制器将继续执行 Step 5 封禁操作。

#### 封禁操作

在 Step 5 中，控制器根据 spec.Duration 字段判断封禁时长：

- 如果 Duration 为空字符串，表示永久封禁，设置 isPermanent = true，并不传递具体时长；
- 如果 Duration 非空，则调用 time.ParseDuration 解析该字符串为具体时长 banSeconds（秒数）；
  - 如果解析失败，则更新状态为失败，并返回错误；

随后调用适配器接口 r.Adapter.Ban(ip, isPermanent, banSeconds) 执行封禁操作。

- 若封禁失败，更新状态为失败，并通过Notify发送错误通知；
- 若封禁成功，更新状态为成功，包括更新时间戳、记录最后的 spec 哈希等信息，并通过通知器发送封禁成功通知。

最后，若为非永久封禁，且未曾自动解封，则启动一个协程异步执行自动解封计划。



对于自动解封函数scheduleAutoUnblock，主要思路就是time.Sleep(传入的时间)阻塞当前协程等待这段时间（即封禁到期时间），然后执行r.Adapter.UnBan(ipblock.Spec.IP)，对于成功/失败，更新状态并通知。

#### 状态更新函数

```go
func (r *IPBlockReconciler) UpdateIPBlockStatus(ctx context.Context, ipblock *opsv1.IPBlock, updateFn func(*opsv1.IPBlock)) (*opsv1.IPBlock, error) {}
```

目的是为了安全地更新Status字段，特别解决并发更新是的版本冲突问题

- 定义一个变量 latest，用来存储每次重试时获取的最新 IPBlock 对象。
- 调用 retry.RetryOnConflict(retry.DefaultBackoff, func() error {...})，该函数会捕获因为版本冲突导致的更新失败，并自动按照默认重试策略重试。
- 在重试函数内部：
  - 先调用 r.Get(...) 从 API Server 获取该 IPBlock 的最新版本，避免基于过时版本更新造成冲突。
  - 调用传入的回调函数 updateFn(&latest)，让调用方修改 latest 对象的 Status 字段。
  - 调用 r.Status().Update(ctx, &latest) 仅更新对象的 Status 子资源（符合 Kubernetes 设计，Status 和 Spec 可分开更新）。
- 如果 Update 出错且是版本冲突错误，RetryOnConflict 会自动重试，直到成功或超过最大重试次数。
- 如果最终失败，记录错误日志。
- 返回最新状态的对象指针和错误。

### 通知模板与动态参数替换

这里主要使用的 Lark 的 card json 模板，将需要替换的部分写成变量形式。

```j
...
"elements": [
              {
                "tag": "markdown",
                "content": "<font color=\"grey\">告警内容</font>\n**${ip}**\n 发起请求过于频繁（1分钟内 **${count}** 次）",
                "i18n_content": {
                  "en_us": "<font color=\"grey\">Alert details</font>\nMobile client crash rate at 5%"
                },
...
```

然后在使用时，动态传入参数即可。（模板类型，传递参数）

```go
if r.Notifier != nil {
			logger.Info("Notifier found, sending ban notification", "ip", ip)
			err := r.Notifier.Notify(ctx, "ban", map[string]string{
				"alarm_time": time.Now().Format("2006-01-02 15:04:05"),
				"ip":         ip,
				"count":      countExtra(ipblock.Spec.Reason), // 这里填实际count
			})
			if err != nil {
				logger.Error(err, "发送封禁通知失败", "ip", ip)
			} else {
				logger.Info("发送封禁通知成功", "ip", ip)
			}
		} else {
			logger.Info("Notifier is nil, skipping ban notification")
		}
```

### ConfigMap热更新

在main.go中watchConfigMap实现。

```go
func watchConfigMap(ctx context.Context, mgr ctrl.Manager, reconciler *controller.IPBlockReconciler) {}
```

1. 通过 mgr.GetCache().GetInformer(ctx, &corev1.ConfigMap{}) 获取 ConfigMap 的 Informer

- Informer 是 controller-runtime 提供的机制，底层封装了 Kubernetes 的缓存和事件监听功能。

- 它会监听集群中 ConfigMap 资源的新增、更新、删除事件，触发相应回调函数。

1. 通过 AddEventHandler 注册事件处理函数，即注册一个"当资源发生变化时需要执行的函数"

- AddFunc：当监听的 ConfigMap 新建时触发。
- UpdateFunc：当监听的 ConfigMap 更新时触发。

1. 具体处理逻辑：

a. 读取并更新关键配置字段

- gatewayHost：网关地址，变更时调用 reconciler.UpdateGatewayHost 动态更新。

- 白名单：调用 config.LoadWhitelistFromConfigMap 加载最新白名单，更新 reconciler.UpdateWhitelist（并发安全更新）。

- 适配器名称（engine 字段）：更新适配器实例。

b. 触发器（Triggers）加载和重启

- 从 ConfigMap 的 trigger 字段加载触发器配置。

- 停止所有旧触发器，重新创建并注册新触发器，最后启动它们。

c. 通知（Notify）配置加载

- 读取通知类型、Webhook URL 和模板。

- 动态创建新的通知实例（如飞书 LarkNotify），替换旧实例，实现通知配置的热切换。

### 并发处理

IPBlock-Operator-Plus在实际使用中可能会出现并发问题，主要会出现以下三种情况：

> 场景举例：
>
> - 单 Trigger 重复报警：Grafana 配置有误，1分钟内对同一 IP 发出10次报警。
> - 多个 Trigger 同时触发：比如 Prometheus 和 Grafana 同时监控 TCP 和 HTTP 流量，触发了对同一 IP 的告警。
> - 多个 IP 并发告警：高峰期系统受到 SYN Flood 攻击，多个 IP 同时发起连接，产生大量告警。

1. 如果单Trigger频繁报警，POST了多个Webhook，可能会出现重复CR资源。

1. 如果有多个Trigger，多个Trigger通知了同一个 IP，也可能出现重复CR资源，或者出现冲突。

1. 多个Trigger通知不同 IP，可能出现资源竞争或冲突。

针对这三种情况，在internal/utils下添加了一些工具专门处理这几种并发问题。

- debouncer.go：定义 Debouncer 接口，拥有一个ShouldAllow(key string) bool方法，用于决定是否允许本次处理。
- lru_debouncer.go：Debouncer 的接口具体实现。利用LRU处理Webhook风暴，避免因为大量 Webhook 通知同一个 IP，导致出现大量重复的 CR 资源。
- gen_name.go：对 IPBlock CR 名进行唯一性标识，避免多个 Trigger 通知同一个 IP，导致出现大量重复的 CR 资源。
- IPLock.go：用于管理 IP 维度的并发锁，避免多个 Trigger 并发冲突或重复创建 CR ，在同一时间只允许一个 Trigger （或一个 goroutine）处理同一个 IP。

#### LRU 防抖机制

确保同一个 IP 只会有一个资源，防止 Webhook 风暴。

> LRU（Least Recently Used）缓存淘汰策略，是一种缓存淘汰机制。它会淘汰最近最少被访问的数据，腾出空间给新数据。
>
> 具体是通过维护两个链表：活跃链表和非活跃链表实现的。
>
> 有新数据访问时，把新数据插入非活跃链表的尾部。
>
> 如果访问了非活跃链表中的数据，就把这个数据放在活跃链表的头部。
>
> 如果访问了活跃链表中的数据，就把这个数据放在活跃链表的头部。
>
> 如果缓存满了，再插入时，淘汰位于链表最后的数据。

逻辑保存在utils/lru_debouncer.go中，这里是对 Debouncer 接口的实现。主要有两个方法：

```go
// 使用"k8s.io/utils/lru"包，生成一个 LRU 防抖器。
func NewLRUDebouncer(size int, ttl time.Duration) *lruDebouncer {}

// 检查当前行为是否合法。
func (d *lruDebouncer) ShouldAllow(ip string) bool {}
```

核心方法是ShouldAllow，这个方法做了下面几件事：

1. 给 cache 加上互斥锁，防止并发请求中对其访问和修改。
2. 在缓存中检查是否有该 IP 的记录
3. 有的话，就检查它上次触发时间 ts 和现在 time.Since(ts) 的间隔。
4. 如果间隔时间小于设定的 TTL，说明是重复触发了，就返回 false，拒绝本次操作。

然后在internal/trigger/grafana.go中应用防抖（在创建 CR 之前）

```go
// 防抖，防止重复创建 CR
if !g.Debouncer.ShouldAllow(ip) {
	logger.Info("[grafana] Skip duplicate IPBlock within TTL", "ip", ip)
	return
}
```

最后在main.go中，注册防抖器。

我们定义了防抖频率规则：

- 对于不同 IP，LRU 中最多保存1000个IP，达到1000个后会自动淘汰最近最少使用的 IP。
- 对于同一个IP，如果最近60s内发生过一次封禁，那么这60s内再次收到该IP的相同请求时，会被防抖识别为重复。

```go
// 选择触发器
func CreateTriggerByConfig(cfg TriggerConfig, mgr ctrl.Manager) trigger.Trigger {
	switch cfg.Name {
	case "grafana":
		return &trigger.GrafanaTrigger{
			Client: mgr.GetClient(),
			Addr:   cfg.Addr,
			Path:   cfg.Path,
			// 1000 个 IP， 60 秒防抖
			Debouncer: utils.NewLRUDebouncer(1000, 60*time.Second),
			IPLocker:  utils.NewIPLock(),
		}
	// TODO 其他触发器 ...
	default:
		return nil
	}
}
```

#### 唯一性 CR 名称

确保同一个 IP 只会有一个资源。

> 既然可能会出现重复的 CR 资源，那我们就通过一些特征来让每个 CR 唯一化，这样多个 Trigger 在上报相同 IP 时，生成的 CR 名也是相同的。K8s不会出现相同名的 CR ，这样就有效避免了大量相同 CR 消耗资源的并发问题。

逻辑保存在utils/gen_name.go中，主要是对 IP 进行 md5 hash，取8位，然后和ipblock-进行拼接。

```go
func GenCRName(ip string) string {
	hash := md5.Sum([]byte(ip))
	return "ipblock-" + hex.EncodeToString(hash[:8]) // 16位
}
```

然后我们在internal/trigger/grafana.go中应用，在创建 CR 之前，先判断 CR 是否已经存在（g.Client.Get() + apierrors.IsNotFound(err)）。

```go
crName := utils.GenCRName(ip)

// 先判断 CR 是否已经存在
var existing opsv1.IPBlock
err := g.Client.Get(context.Background(), client.ObjectKey{
	Name:      crName,
	Namespace: "default",
}, &existing)
if err == nil {
	logger.Info("[grafana] IPBlock already exists, skip creation", "ip", ip)
	return
}
	
// 如果 NotFound，继续创建
if err != nil && !apierrors.IsNotFound(err) {
	logger.Error(err, "[grafana] Error checking IPBlock existence", "ip", ip)
	return
}
```

#### IP锁机制

确保同一时间只允许一个 Trigger （或一个 goroutine）处理同一个 IP 的封禁逻辑。

> 假设同一时刻，这两个 trigger 几乎同时收到了针对 IP 1.2.3.4 的 webhook：
>
> - TriggerA 开始处理：打算创建一个 IPBlock 资源
> - TriggerB 也收到相同 IP 的告警，也想创建一个同样的 IPBlock
>
> 如果没有加锁：
>
> - 两个 trigger 可能几乎同时进入了 Client.Create(...)
> - 结果 Kubernetes 可能报错 AlreadyExists
> - 状态可能冲突，日志也会乱

逻辑保存在utils/IPLock.go中，主要做三件事：

```go
type IPLock struct {
	global sync.Mutex // 保护 locks map
	locks  map[string]*sync.Mutex // 全局锁，用来保护 locks 这个 map，防止并发读写它时发生竞态。
}
// 新建一个锁管理器
func NewIPLock() *IPLock {}

// 获取某个 IP 的锁
func (i *IPLock) Lock(ip string) {}

// 释放某个 IP 的锁
func (i *IPLock) Unlock(ip string) {}
```

核心是Lock，它做了这几件事：

1. 新建一个全局锁，防止多个 goroutine 访问 map 出现问题。
2. 查看该 IP 是否有对应的锁，没有就新建一个锁，并加入map中。
3. 再释放全局锁。
4. 最后调用具体 IP 的lock.Lock()，阻塞等待获取锁。

然后应用在internal/trigger/grafana.go，对预警循环时，会循环获取到每个 IP，这里我们使用匿名函数，为了确保每个 IP 的锁在处理完成后能及时释放，避免因 defer 延迟释放导致的锁阻塞，我们为每个 IP 的处理逻辑包裹一个匿名函数：

```go
// 避免并发时的竞争和死锁
func(ip string) {
	g.IPLocker.Lock(ip)
    defer g.IPLocker.Unlock(ip)
	...
}(ip)
```

这样设计的好处是每轮循环的 defer 都会在当前匿名函数结束时释放锁，避免多个 IP 的处理互相阻塞，保证并发安全。

## 插件扩展

项目的组件放在internal/下

- controller（控制器）：负责核心业务逻辑的调度和协调，监听 Kubernetes 中 IPBlock 自定义资源的变化，维护和管理其生命周期。
- engine（封禁后端）：负责封禁命令的实际执行。
- notify（通知机制）：负责将封禁、解封等事件以多种方式通知给运维人员或其他系统。
- trigger（触发器）：事件触发中心，负责监听外部告警系统或业务事件（如 Grafana Alert等），并根据 Alert 策略触发相关封禁操作，自动创建 IPBlock CR，且支持自动解封。
- policy（封禁策略）：定义 IP 封禁的判定规则和执行策略。

### engine

#### 介绍

封禁后端均被抽象为API，IPBlock-Operator-Plus通过调用API来实现实际的封禁行为。

目前通过engine/control.py来实现API，接口列表如下：

| 接口     | 方法 | 说明                                          |
| -------- | ---- | --------------------------------------------- |
| /limit   | GET  | iptables限流接口                              |
| /unlimit | GET  | iptables解限流接口                            |
| /limits  | GET  | iptables查看当前限流情况                      |
| /update  |      | XDP封禁端口                                   |
| /remove  |      | XDP解封禁端口                                 |
| /ban     | GET  | （弃用）可从nginx日志查，返回超过一定次数的IP |
| /execute | GET  | （弃用）接收IP，对其执行XDP封禁               |

engine支持列表：

- XDP：依赖于[evilsp/xdp_banner: 一个简单的 XDP 小程序，用于 BAN IP](https://github.com/evilsp/xdp_banner)

- iptables：依赖于Linux工具iptables。

  规则为 IP 每分钟最多发起10个新连接（可突发20次），否则DROP

  ```go
  iptables -A INPUT -s <IP> -p tcp --dport <TARGET_PORT> \
    -m state --state NEW \
    -m hashlimit --hashlimit 10/min --hashlimit-burst 20 \
    --hashlimit-mode srcip --hashlimit-name limit_<IP_REPLACED> \
    -j ACCEPT
  
  iptables -A INPUT -s <IP> -p tcp --dport <TARGET_PORT> -j DROP
  ```

#### 扩展开发指南

engine定义了接口，新adapter只需要实现这两个方法即可。

```go
type Adapter interface {
	// Ban 对某个 IP 发起封禁
	// ip: 要封禁的 IP 地址
	// isParmanent: 是否永久封禁（true 表示永久）
	// durationSeconds: 封禁时长（单位：秒，仅在临时封禁时生效）
	Ban(ip string, isParmanent bool, durationSeconds int) (string, error)

	// UnBan 解封某个 IP
	UnBan(ip string) (string, error)
}
```

然后在NewAdapter注册对应的adapter

```go
func NewAdapter(name, gatewayHost string) Adapter {
	switch name {
	case "xdp":
		return &XDPAdapter{GatewayHost: gatewayHost}
	case "iptables":
		return &IptablesAdapter{GatewayHost: gatewayHost}
	default:

		return &XDPAdapter{GatewayHost: gatewayHost}
	}
}
```

接着在controller.py中要实现对应的API，最后在configmap中engine字段指定对应的adapter名即可。

### notify

#### 介绍

notify支持列表：

- lark：飞书，通过机器人来进行通知

#### 扩展开发指南

notify定义了接口，新notify只需要实现这个接口即可。

```go
// Notifier 是通知发送接口的抽象定义，
// 各种通知后端（如飞书、邮件、Webhook等）都应实现此接口。
type Notifier interface {
	// Notify 触发一个通知事件。
	//
	// ctx: 标准的 context 对象，用于控制取消、超时等上下文行为。
	// eventType: 事件类型，通常为 "ban"、"resolve"、"common" 等预定义事件名。
	// vars: 用于填充通知模板的变量键值对，例如 map["ip"]="1.2.3.4"，map["reason"]="攻击频繁"。
	//
	// 返回值:
	//   - 成功时返回 nil
	//   - 失败时返回 error，用于上层重试或告警记录
	Notify(ctx context.Context, eventType string, vars map[string]string) error
}
```

以lark为例，在lark.go中实现两个方法

```go
// NewLarkNotify 创建一个 LarkNotify（飞书通知器）实例。
//
// 参数:
//   - webhookURL: 飞书群机器人的 Webhook 地址。
//   - templatePaths: 各类通知类型所用的 card 模板路径映射（如 map["ban"]="templates/ban.json"）。
//
// 返回值:
//   - 成功: 返回 LarkNotify 实例。
//   - 失败: 返回 error，通常是模板解析失败或配置不合法。
func NewLarkNotify(webhookURL string, templatePaths map[string]string) (*LarkNotify, error) {}
// Notify 实现 Notifier 接口，
// 根据事件类型和变量构造飞书卡片消息，并发送至 webhook。
//
// 参数:
//   - ctx: 请求上下文，用于控制超时和取消等。
//   - eventType: 事件类型（如 "ban"、"resolve"、"error"）。
//   - vars: 模板变量键值对，例如 {"ip": "1.2.3.4", "reason": "恶意连接"}。
//
// 返回值:
//   - 成功: 返回 nil。
//   - 失败: 返回 error，表示通知失败。
func (l *LarkNotify) Notify(ctx context.Context, eventType string, vars map[string]string) error {}
```

最后在main.go中watchConfigMap中完善loadNotify。

```go
// 以飞书为例
// 加载通知中心
		loadNotify := func(cm *corev1.ConfigMap) {
			notifyType := cm.Data["notifyType"]
			webhookURL := cm.Data["notifyWebhookURL"]

			templates := make(map[string]string)
			for k, v := range cm.Data {
				if strings.HasPrefix(k, "notifyTemplate_") {
					eventType := strings.TrimPrefix(k, "notifyTemplate_")
					templates[eventType] = v
				}
			}

			if notifyType == "lark" && webhookURL != "" && len(templates) > 0 {
				larkNotify, err := lark.NewLarkNotify(webhookURL, templates)
				if err != nil {
					log.Log.Error(err, "Failed to create LarkNotify instance")
					reconciler.Notifier = nil
					return
				}
				reconciler.Notifier = larkNotify
				log.Log.Info("LarkNotify has been initialized")
				return
			}

			//TODO 其他通知方式...
```

### trigger

#### 介绍

trigger支持列表：

- grafana：可以 Grafana Alert 联动，通过 Webhook 进行触发。

#### 扩展开发指南

> 注意新 trigger 在开发时需要考虑并发问题。具体实现可查看 grafana.go 中的处理逻辑，也可参考核心功能模块开发文档中并发处理的逻辑介绍。

trigger同样定义了接口，新trigger只需要实现接口即可。

```go
// Trigger 是封禁事件的触发器接口，Start 启动监听任务，Stop 停止监听任务
type Trigger interface {
	Name() string
	Start(ctx context.Context) error
	Stop(ctx context.Context) error
}
```

trigger在manager.go中实现了StartAll和StopAll，会启动在configmap中指定的所有trigger。

```go
  triggers:                                          # 触发器，目前仅支持 Grafana
    - name: grafana
      addr: ":8090"
      path: "/trigger/grafana"
    - name: your_trigger
      other: xxx
```

trigger自定义的配置字段，在main.go中进行配置

```go
// Grafana Trigger 示例
type TriggerConfig struct {
	Name string `yaml:"name"`
	Addr string `yaml:"addr,omitempty"`
	Path string `yaml:"path,omitempty"`
}

// 解析 trigger 字符串为 YAML 列表
func parseTriggers(yamlStr string) ([]TriggerConfig, error) {
	var triggers []TriggerConfig
	err := yaml.Unmarshal([]byte(yamlStr), &triggers)
	if err != nil {
		return nil, err
	}
	return triggers, nil
}

// 选择触发器
func CreateTriggerByConfig(cfg TriggerConfig, mgr ctrl.Manager) trigger.Trigger {
	switch cfg.Name {
	case "grafana":
		return &trigger.GrafanaTrigger{
			Client: mgr.GetClient(),
			Addr:   cfg.Addr,
			Path:   cfg.Path,
			// 1000 个 IP， 60 秒防抖
			// LRU中最多保存1000个IP，达到1000个后会自动淘汰最近最少使用的IP
			// 对于同一个IP，如果最近60s内发生过一次封禁，那么这60s内再次收到该IP的相同请求时，会被防抖识别为重复
			Debouncer: utils.NewLRUDebouncer(1000, 60*time.Second),
			IPLocker:  utils.NewIPLock(),
		}
	// TODO 其他触发器 ...
	default:
		return nil
	}
}
```

### policy

policy支持功能列表：

- whitelist：白名单机制

policy实现比较灵活，下面描述白名单机制的实现流程。

1. 在policy/watchlist.go中实现三个函数

   ```go
   // 白名单机制
   // 单IP白名单
   // CIDR白名单
   // 标签匹配
   
   type Whitelist struct {
   	ipNets []*net.IPNet // IPNet的指针切片，保存CIDR网段
   	ips    []net.IP     // 精确IP白名单
   }
   
   // 新建白名单
   func NewWhitelist(ipList []string) *Whitelist {}
   
   // 判断目标IP是否在白名单中
   func (w *Whitelist) IsWhitelisted(ip string) bool {}
   
   // 打印所有白名单内容
   func (w *Whitelist) StringSlice() []string {}
   ```

1. 在config/loader.go中实现一个LoadWhitelistFromConfigMap，用于加载定义在configmap中的白名单IP。
2. 在main.go的watchConfigMap中来监听白名单的新建和更新情况。
