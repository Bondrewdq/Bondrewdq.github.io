---
title: 14-集群网络与 DNS 解析深潜
tags: []
id: '255'
categories:
  - - k8s 实验
date: 2026-03-11 09:11:36
---

#### 核心心智模型

Service 只是空壳，Endpoints 才是灵魂

初学者一般会认为流量是发给 Service，然后再由 Service 发给 Pod，这是错的。

在 Linux 网络栈的底层，Service 分配的 ClusterIP 其实是一个"假 IP"，它连真实的网卡都没有。

真正的流程是：

1.  CoreDNS 负责把域名翻译成这个虚拟的 ClusterIP。
2.  K8s 控制台会在后台默默维护一个叫 Endpoints 的列表，里面记录了真实 Pod 的物理 IP。
3.  节点上的 kube-proxy 会读取 Endpoints 列表，并在 Linux 内核写入底层的转发规则。流量打到虚拟 IP 后，在操作系统底层就被直接 NAT 转发到了真实的 Pod IP 上。

#### 实验过程

##### 实验准备

我们将部署一个目标 Web 服务，以及一个专门用于网络抓包和测试的 Pod。

1.  创建目标服务
    
    ```
    kubectl create deployment web-target --image=nginx
    kubectl expose deployment web-target --port=80 --name=web-svc
    ```
    
2.  部署网络排查 Pod
    
    ```
    kubectl run net-tools --image=nicolaka/netshoot -- sleep 3600
    ```
    
    (等待两个 Pod 都进入 Running 状态)
    

##### 阶段一：DNS 解析深潜

1.  进入网络排查 Pod 内部：
    
    ```
    kubectl exec -it net-tools -- bash
    ```
    
2.  使用 nslookup 查询 Service 的域名
    
    ```
    nslookup web-svc
    ```
    
    预期输出：会看到不仅解析出了 web-svc 的 ClusterIP，还打印出了完整限定域名（FQDN)：web-svc.default.svc.cluster.local。
    
3.  查看容器的 DNS 配置文件
    
    为什么只敲 web-svc 这个名字就能通网络？
    
    ```bash
    cat /etc/resolv.conf
    ```
    
    会看到 ：
    
    ```
    net-tools:~# cat /etc/resolv.conf
    search default.svc.cluster.local svc.cluster.local cluster.local
    nameserver 10.96.0.10
    options ndots:5
    ```
    
    *   `search default.svc.cluster.local svc.cluster.local` 表示填补的后缀，当请求 `web-svc` 时，Linux 底层会自动帮你把这些后缀补齐，然后发给 `nameserver`（这就是 CoreDNS 的 IP）。
    *   `ndots` 决定了 DNS 什么时候去尝试补全域名后缀。如果你查询的域名中，包含的“点号”数量 **小于** `ndots` 的设定值，系统会认为这不是一个完整的域名（FQDN），会**优先**尝试加上 `search` 列表里的后缀去查。
    
    _退出侦察机，回到宿主机终端)_
    
    Bash
    
    ```
    exit
    ```
    

##### 阶段二：Endpoints 机制与 “Connection Refused”案发现场

这是 CKA 最经典的排错场景：**开发人员抱怨“域名能通，但连接被拒绝”。** 我们来手动制造一场惨案。

**1.查看当前健康状态**

```

kubectl get endpoints web-svc
```

**2\. 制造破坏（修改 Service 标签）：**

我们将 Service 寻找后端 Pod 的“规则”故意改错。

```

kuebctl set selector svc web-svc wrong:label
```

这里之所以不用`patch` 命令，这是因为 Kubernetes 默认的合并策略是**增量更新**。在 Kubernetes 的 Selector 逻辑中，多个标签之间是 **“AND（且）”** 的关系。

查看修改后的 web-svc Endpoints 的 selector 标签：

```

kubectl get svc web-svc -o jsonpath='{.spec.selector}'
```

**3\. 确认破坏结果**

```

kubectl get endpoints web-svc
```

**预期现象：** `ENDPOINTS` 变成了 `<none>`！此时

**4\. 模拟客户端访问，重现经典报错**

```

kunectl exec -it net-tools -- curl -m 3 web-svc
```

**预期现象：** `curl: (7) Failed to connect to web-svc port 80: Connection refused`

深度解析：为什么是 Conection refused?

当执行 curl 时，DNS 依然完美地把 web-svc 解析成 ClusterIP，因为 Service 还在。但当网络包带着这个 ClusterIP 进入 Linux 网络栈时，`iptables` 发现这个虚拟 IP 背后没有任何物理 Endpoints 支撑，于是内核直接向客户端返回了 TCP RST 包，导致连接被拒绝。

##### 阶段三：抢救与恢复

在考场上遇到 `Connection refused` 或者访问不通，第一反应永远是：**检查 Service 的 Selector 和 Pod 的 Label 是否完美匹配！**

**1\. 查看 Pod 的真实标签：**

```

kubectl get pods --show-labels  grep web-target
```

_(假设输出显示标签是 `app=web-target`)_

**2\. 修复 Service**

你可以用 `kubectl edit svc web-svc` 把 selector 改回来，或者用命令行直接打补丁：

```

kubectl set selector svc web-svc app=web-target
```

**3\. 见证 Endpoints 归位与网络恢复**

```

kubectl get endpoints web-svc
kubectl exec -it net-tools -- curl -s -o /dev/null -w "%{http_code}\n" web-svc
```

(看到 `200`，说明流量重新打通！)

##### 清理战场

```

kubectl delete deployment web-target
kubectl delete svc web-svc
kubectl delete pod net-tools
```