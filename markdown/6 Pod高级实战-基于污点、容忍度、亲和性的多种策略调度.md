[TOC]



# 1. 基本概念

## 1.1 污点是什么？

- **污点（Taint）**：是应用于节点上的一种属性，它使得节点拒绝将 Pod 调度到该节点上，除非 Pod 明确声明能够容忍这些污点。通过设置污点，可以让节点被用于特定类型的 Pod， 比如将有特殊硬件要求或资源消耗大的 Pod 调度到指定节点。



## 1.2 容忍度是什么？

- **容忍度（Toleration）**：是 Pod 的一个特性，用于声明该 Pod 可以被调度到带有特定污点的节点上。一个 Pod 可以有多个容忍度，以此来匹配节点上的不同污点 。



## 1.3 亲和性是什么？

- **亲和性（Affinity）**：分为节点亲和性（Node Affinity）和 Pod 亲和性（Pod Affinity）。节点亲和性让用户能够将 Pod 调度到满足特定条件的节点上；Pod 亲和性则可以让 Pod 与其他 Pod 部署在相同拓扑域（如同一节点、同一可用区等）。



## 1.4 反亲和性是什么？

- **反亲和性（Anti - Affinity）**：同样分为节点反亲和性和 Pod 反亲和性。节点反亲和性使 Pod 避免被调度到满足某些条件的节点上；Pod 反亲和性则让 Pod 避免与其他 Pod 部署在相同拓扑域，常用于实现高可用性，防止多个关键 Pod 同时部署在同一故障域 。















# 2. 不同类型及核心差异

## 2.1 污点（Taint）的类型

通过 `effect` 字段定义，决定节点如何拒绝 Pod 调度，有 3 种核心类型：

| 类型（effect 值）  | 作用规则                                                     | 典型场景                    |
| ------------------ | ------------------------------------------------------------ | --------------------------- |
| `NoSchedule`       | 仅拒绝**未声明容忍**的新 Pod 调度，已运行的 Pod 不受影响     | 常规资源隔离（如 GPU 节点） |
| `PreferNoSchedule` | 尽量不调度未容忍的 Pod，但集群资源不足时仍可调度（“软” 限制） | 非关键资源的优先隔离需求    |
| `NoExecute`        | 不仅拒绝新 Pod 调度，已运行的 Pod 若不满足容忍度也会**被驱逐** | 节点故障、安全策略强制生效  |



## 2.2 容忍度（Toleration）的类型

通过 `operator` 字段定义匹配逻辑，主要 2 种类型：

| 类型（operator 值） | 作用规则                                                     | 典型场景                                     |
| ------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| `Equal`             | 严格匹配 `key=value`，需同时指定 `key` 和 `value`            | 精准匹配特定污点（如 `gpu=true`）            |
| `Exists`            | 仅匹配 `key` 存在（无需 `value`），表示容忍所有含该 `key` 的污点 | 通用容忍需求（如容忍所有 `node-taint` 类型） |



## 2.3 亲和性（Affinity）的类型

分**节点亲和性**和**Pod 亲和性**，各自又分 “硬需求” 和 “软需求”：

**1. 节点亲和性（Node Affinity）**

| 类型（调度规则） | 字段定义                                          | 作用规则                                                    |
| ---------------- | ------------------------------------------------- | ----------------------------------------------------------- |
| **硬需求**       | `requiredDuringSchedulingIgnoredDuringExecution`  | 必须满足条件，否则 Pod 无法调度（类似 “必须选”）            |
| **软需求**       | `preferredDuringSchedulingIgnoredDuringExecution` | 优先满足条件，若无法满足仍可调度到其他节点（类似 “尽量选”） |



**2. Pod 亲和性（Pod Affinity）**

| 类型（调度规则） | 字段定义                                          | 作用规则                                                     |
| ---------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| **硬需求**       | `requiredDuringSchedulingIgnoredDuringExecution`  | Pod 必须与目标 Pod 同拓扑域（如同一节点、可用区），否则无法调度 |
| **软需求**       | `preferredDuringSchedulingIgnoredDuringExecution` | 优先与目标 Pod 同拓扑域，无法满足时可调度到其他域            |



## 2.4 反亲和性（Anti-Affinity）的类型

分**节点反亲和性**和**Pod 反亲和性**，同样有 “硬需求” 和 “软需求”：

**1. 节点反亲和性（Node Anti-Affinity）**

| 类型（调度规则） | 字段定义                                          | 作用规则                                                     |
| ---------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| **硬需求**       | `requiredDuringSchedulingIgnoredDuringExecution`  | Pod 必须**不调度**到满足条件的节点，否则无法部署（类似 “必须避开”） |
| **软需求**       | `preferredDuringSchedulingIgnoredDuringExecution` | 优先不调度到满足条件的节点，资源不足时仍可调度（类似 “尽量避开”） |

**2. Pod 反亲和性（Pod Anti-Affinity）**

| 类型（调度规则） | 字段定义                                          | 作用规则                                                     |
| ---------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| **硬需求**       | `requiredDuringSchedulingIgnoredDuringExecution`  | Pod 必须**不与**目标 Pod 同拓扑域，否则无法部署（如多副本分散节点） |
| **软需求**       | `preferredDuringSchedulingIgnoredDuringExecution` | 优先不与目标 Pod 同拓扑域，资源不足时仍可共域（“尽量分散”）  |



## 2.5 核心差异总结

| 维度         | 污点（Taint）               | 容忍度（Toleration）             | 亲和性（Affinity）                       | 反亲和性（Anti-Affinity）                |
| ------------ | --------------------------- | -------------------------------- | ---------------------------------------- | ---------------------------------------- |
| **作用对象** | 节点（Node）                | Pod                              | Pod（调度到节点 / Pod 同域）             | Pod（调度到节点 / Pod 不同域）           |
| **核心逻辑** | 节点拒绝 Pod 调度（“排斥”） | Pod 声明允许被排斥（“接受排斥”） | Pod 主动贴近节点 / Pod（“吸引”）         | Pod 主动远离节点 / Pod（“排斥”）         |
| **类型控制** | 通过 `effect` 定义拒绝强度  | 通过 `operator` 定义匹配逻辑     | 通过 “硬 / 软需求” + 节点 / Pod 维度划分 | 通过 “硬 / 软需求” + 节点 / Pod 维度划分 |

简单说：

- **污点 + 容忍度** = 实现节点对 Pod 的 “准入控制”（节点说 “不”，Pod 说 “我能忍” 才能调度）；
- **亲和性** = 让 Pod “主动贴近” 目标（节点或其他 Pod）；
- **反亲和性** = 让 Pod “主动远离” 目标（节点或其他 Pod）。

这些类型组合，可覆盖大厂复杂调度场景（如资源独占、高可用分散、性能优化等）。















# 3. 实战项目

## 3.1 场景 1：电商大促 - 污点（Taint）+ 容忍度（Toleration）

**需求**：GPU 节点仅允许「深度学习推理 Pod」调度，普通 Pod 拒绝调度。

**1. 给节点打污点（操作步骤）**

```yaml
# 1. 找到目标节点（假设节点名：k8s-node-gpu-01）
kubectl get nodes  

# 2. 给节点打污点（effect=NoSchedule：拒绝无容忍度的 Pod）
kubectl taint nodes k8s-node-gpu-01 gpu-node=true:NoSchedule
```

**2. Pod 配置容忍度（YAML）**

```shell
apiVersion: v1
kind: Pod
metadata:
  name: deep-learning-pod
  labels:
    app: image-search  # 业务标签
spec:
  containers:
  - name: inference-container
    image: inference-service:v1.0  # 推理服务镜像   registry.cn-shenzhen.aliyuncs.com/amgs/nginx:latest
  # 关键：容忍 GPU 节点的污点
  tolerations:
  - key: "gpu-node"          # 匹配污点的 key
    operator: "Equal"        # 匹配方式：等于
    value: "true"            # 匹配污点的 value
    effect: "NoSchedule"     # 匹配污点的生效策略
```

 **3. 验证（操作步骤）**

```yaml
# 部署 Pod
kubectl apply -f deep-learning-pod.yaml  

# 检查 Pod 调度结果（应调度到 k8s-node-gpu-01）
kubectl get pods -o wide  
```



## 3.2 场景 2：游戏部署 - 节点亲和性（Node Affinity）

**需求**：「登录 / 存档 Pod」优先调度到 **SSD 硬盘节点**（标签：`disk-type=ssd`）。

**1. 给节点打标签（操作步骤）**

```shell
# 给目标节点打标签（假设节点名：k8s-node-ssd-01）
kubectl label nodes k8s-node-ssd-01 disk-type=ssd  
```

**2. Pod 配置节点亲和性（YAML）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: game-login-pod
  labels:
    app: game-login  # 业务标签
spec:
  containers:
  - name: login-container
    image: game-login:v2.0  # 登录服务镜像
  # 关键：节点亲和性配置
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 调度时强制要求
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk-type       # 匹配节点标签 key
            operator: In         # 匹配方式：在 values 列表中
            values:
            - ssd                # 目标节点标签 value
```

**3. 验证（操作步骤）**

```shell
kubectl apply -f game-login-pod.yaml  
kubectl get pods -o wide  # 检查是否调度到 ssd 节点
```



## 3.3 **场景 3：社交平台 - Pod 亲和性（Pod Affinity）**

**需求**：「消息推送 Pod」与「消息存储 Pod」**同可用区部署**（减少网络延迟）。

**1. 消息存储 Pod（提前部署，YAML 示例）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: message-storage-pod
  labels:
    app: message-storage  # 关键标签，供亲和性匹配
spec:
  containers:
  - name: storage-container
    image: message-storage:v1.0  
```

**2. 消息推送 Pod 配置亲和性（YAML）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: message-push-pod
  labels:
    app: message-push  # 业务标签
spec:
  containers:
  - name: push-container
    image: message-push:v1.0  
  # 关键：Pod 亲和性配置
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 调度时强制要求
      - labelSelector:
          matchLabels:
            app: message-storage  # 匹配「消息存储 Pod」的标签
        topologyKey: failure-domain.beta.kubernetes.io/zone  # 同可用区
```

**3. 验证（操作步骤）**

```shell
kubectl apply -f message-storage-pod.yaml  
kubectl apply -f message-push-pod.yaml  

# 检查两个 Pod 是否在同一可用区（查看 node 的 zone 标签）
kubectl get pods -o wide  
```



## 3.4 **场景 4：在线教育 - Pod 反亲和性（Pod Anti-Affinity）**

**需求**：「直播核心 Pod」**分散部署在不同节点**（避免单点故障）。

**1. Pod 配置反亲和性（YAML）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-core-pod
  labels:
    app: live-core  # 关键标签，供反亲和性匹配
spec:
  containers:
  - name: live-container
    image: live-core:v1.0  
  # 关键：Pod 反亲和性配置
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 调度时强制要求
      - labelSelector:
          matchLabels:
            app: live-core  # 匹配同类 Pod 的标签
        topologyKey: kubernetes.io/hostname  # 不同节点（主机名）
```

**2. 验证（操作步骤）**

```shell
# 部署多个 Pod（模拟高可用场景）
kubectl apply -f live-core-pod.yaml  
kubectl apply -f live-core-pod-2.yaml  

# 检查 Pod 调度结果（应分布在不同节点）
kubectl get pods -o wide  
```



## 3.5 总结

**核心逻辑总结**

| 概念             | 作用                            | 典型场景                    |
| ---------------- | ------------------------------- | --------------------------- |
| **污点**         | 节点「拒绝」无容忍度的 Pod 调度 | 专属资源（GPU、高性能节点） |
| **容忍度**       | Pod「允许」调度到带污点的节点   | 匹配污点，占用专属资源      |
| **节点亲和性**   | Pod 优先调度到「特定标签节点」  | 依赖硬件（SSD、GPU）的服务  |
| **Pod 亲和性**   | Pod 优先与「目标 Pod 同拓扑域」 | 减少网络延迟（存储 + 计算） |
| **Pod 反亲和性** | Pod 避免与「同类 Pod 同拓扑域」 | 高可用（分散部署，防故障）  |

通过 YAML 配置 + `kubectl` 操作，可精准控制 Pod 调度逻辑，适配大厂复杂业务需求（如资源隔离、性能优化、高可用）。