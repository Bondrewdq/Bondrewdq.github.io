---
title: 08-NetworkPolicy
tags: []
id: '243'
categories:
  - - k8s 实验
date: 2026-03-11 09:03:06
---

NetworkPolicy 必须依赖底层网络插件（CNI）的支持。比如 Flannel 就不支持网络策略。

这里集群使用的网络插件是 Calico ，支持 NetworkPolicy。

Kubernetes 网络“白名单“机制：

*   在没有任何 NetworkPolicy 情况下，集群内部网络默认全通（Default Allow)。任何 Pod 都能随意访问其他 Pod。
*   一旦创建 NetworkPolicy，并且通过 PodSelector 选中某个 Pod，这个 Pod 的网络就会立刻变成默认拒绝，只有在 Policy 中明确写明允许的流量（白名单）才能进出。

#### 实验目的

创建一个模拟数据库（db），然后创建两个客户端。我们通过 NetworkPolicy 拦截其中一个客户端请求，放行另一个。

#### 实验过程

##### 阶段一：搭建测试靶场

**1、创建独立命名空间和目标服务**

```

## 创建专属测试命名空间
kubectl create ns np-test

## 运行一个模拟数据库（带有 app=db 标签），并直接暴露 80 端口
kubectl run db --image=nginx --labels="app=db" -n np-test --expose --port=80
```

–expose 会创建 Service，依据 --port 的内容进行端口映射。会创建一个 **ClusterIP** 类型的 Service，将 Service 的 80 端口映射到 Pod 的 80 端口。

2、创建两个模拟客户端

一个带有 access=ture 的标签，另一个带有 access=false 的标签。

```bash
## 允许访问的客户端 
kubectl run allow-client --image=busybox --labels="access=true" -n np-test -- sleep 3600

## 被拒绝访问的客户端
kubectl run deny-client --image=busybox --labels="access=false" -n np-test -- sleep 3600
```

3、验证默认状态（全通）

此时没有策略，两个客户端应该都能访问 db。

```bash
## 从 allow-client 访问
kubectl exec allow-client -n np-test -- wget -qO- --timeout=3 db

## 从 deney-client 访问
kubectl exec deney-client -n np-test -- wget -qO- --timeout=3 db
```

##### 阶段二：施加 NetworkPolicy

现在我们写一个策略，只允许带有 access=true 标签的 Pod 访问 app=db 的 80 端口。

1、创建策略文件 np-db.yaml

可以直接官网搜索 NetworkPolicy 查找模板进行修改

`np-db.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-protect
  namespace: np-test
spec:
  podSelector:
    matchLabels:
      app: db           # 1. 选中要保护的目标 (选中即开启默认拒绝)
  policyTypes:
  - Ingress             # 2. 声明我们要控制入站流量 (Ingress)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"  # 3. 白名单：允许哪个标签的 Pod 进来
    ports:
    - protocol: TCP
      port: 80            # 4. 白名单：允许访问哪个端口
```

2、应用策略

```

kubectl apply -f np-db.yaml
```

##### 阶段三：见证策略生效

1、验证被许可的客户端

```bash
kubectl exec allow-client -n np-test -- wget -qO- --timeout=3 dbKubectl 
```

依然可以连接

2、验证被拦截的客户端

```bash
kubectl exec deny-client -n np-test -- wget -qO- --timeout=3 db
```

**预期结果：** 命令会卡住 3 秒钟，然后报 `wget: download timed out` 或者 `Connection timed out`。这就是 NetworkPolicy 正在底层将数据包默默丢弃。

##### 清理

```bash
kubectl delete ns np-test
```

#### 相关知识

**Namespace 隔离问题：** 如果你在 `from` 里面只写了 `podSelector`，它**默认只会在同一个 Namespace 里找 Pod**。如果考试要求允许另一个 Namespace 的 Pod 访问，你必须同时加上 `namespaceSelector`。

**多重条件（AND vs OR）：**

*   如果 `podSelector` 和 `namespaceSelector` 写在**同一个连字符 `-` 下面**，这是 **AND（且）** 的关系。
*   如果写在**不同的连字符 `-` 下面**，这是 **OR（或）** 的关系。这是考试最爱挖的坑，缩进错一个空格，意义完全不同！