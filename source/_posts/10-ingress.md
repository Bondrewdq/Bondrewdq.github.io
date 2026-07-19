---
title: 10-Ingress
tags: []
id: '247'
categories:
  - - k8s 实验
date: 2026-03-11 09:06:00
---

#### 核心心智模型：Ingress 的技术架构

1.  **Pod (工作负载)**：实际处理业务请求的容器，拥有动态分配的内部 IP。
    
2.  **Service (ClusterIP)**：内部的四层负载均衡器。它提供一个固定的内部虚拟 IP，并将流量轮询转发给后端的 Pod 集合。外部网络无法直接路由到 ClusterIP。
    
3.  **Ingress Controller (反向代理网关)**：本质上是一个运行在集群内的守护进程 Pod（通常基于 Nginx、Envoy 或 Traefik）。它通过 NodePort 或 LoadBalancer 的方式将自己暴露给集群外部，负责接收所有外部的 HTTP/HTTPS 入站流量。
    
4.  **Ingress Resource (路由规则对象)**：这是你通过 YAML 创建的 Kubernetes API 对象。它本身不处理任何流量，仅仅是一份“配置清单”，定义了域名（Host）和路径（Path）到对应 Service 的映射关系。
    

Ingress Controller 会持续监听 API Server 中 Ingress Resource 的变化，并将其动态解析为网关（如 `nginx.conf`）的底层路由配置。外部请求到达 Controller 后，Controller 根据配置直接将流量反向代理到对应的目标 Service。

#### 实验目标：基于域名的七层流量路由

部署 Nginx Ingress Controller，运行两个不同的后端 Web 服务，并配置基于域名的路由规则。

#### 实验过程

##### 阶段一：部署 Ingress Controller

Kubeadm 初始化的裸机集群默认没有 Controller。我们需要手动部署官方的 Nginx Ingress Controller，并通过 NodePort 将其暴露。

**1\. 应用官方裸机版 Controller 部署文件**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```

**2\. 确认 Controller 状态**

```bash
kubectl get pods -n ingress-nginx
```

_(确认为 Running 状态后再进行下一步)_

**3\. 获取暴露的 NodePort 端口**

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

_(在输出的 `PORT(S)` 列中，找到 `80` 对应的五位数端口号，例如 `80:31234/TCP`。请记住这个数字，这是外部流量的真实入口)_

* * *

##### 阶段二：部署后端业务与 Service

创建两个独立的 Deployment，并分别暴露为内部的 ClusterIP Service。

**1\. 部署 app1 (Nginx)**

```bash
kubectl create deployment app1 --image=nginx
kubectl expose deployment app1 --port=80 --name=svc1
```

**2\. 部署 app2 (Httpd)**

```bash
kubectl create deployment app2 --image=httpd
kubectl expose deployment app2 --port=80 --name=svc2
```

* * *

##### 阶段三：创建 Ingress 路由规则

定义路由逻辑：访问 `app1.local` 转发到 `svc1`，访问 `app2.local` 转发到 `svc2`。

**1\. 编写规则文件 `ingress-route.yaml`**

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx   # <--- 明确指定使用 nginx 控制器
  rules:
  - host: "app1.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc1
            port:
              number: 80
  - host: "app2.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc2
            port:
              number: 80
```

**2\. 应用并查看资源**

```bash
kubectl apply -f ingress-route.yaml
kubectl get ingress demo-ingress
```

如果你看到 `ADDRESS` 列出现了一个 IP（可能需要等几秒钟），或者你看不到警告信息，说明 Nginx 控制器终于把这套规则加载到自己的内存里了。\*\*\*\*

* * *

##### 阶段四：外部访问验证

由于我们没有配置真实的 DNS 解析，我们将使用 `curl` 命令配合 `-H "Host: ..."` 参数，强制在 HTTP 请求头中注入域名，以此来测试 Controller 的路由分发能力。

_(请将 `<NodePort>` 替换为阶段一获取的具体端口号，将 `<NodeIP>` 替换为集群中任意节点的物理 IP)_

**1\. 测试路由到 app1：**

```bash
curl -H "Host: app1.local" http://<NodeIP>:<NodePort>
```

_(预期结果：返回 Nginx 默认欢迎页 HTML)_

**2\. 测试路由到 app2：**

```bash
curl -H "Host: app2.local" http://<NodeIP>:<NodePort>
```

_(预期结果：返回 Httpd 默认页 `<html><body><h1>It works!</h1></body></html>`)_

**3\. 测试未定义的路由（越权/异常拦截）：**

```bash
curl -H "Host: unknown.local" http://<NodeIP>:<NodePort>
```

_(预期结果：返回 `404 Not Found`，由 Nginx Ingress Controller 拦截并响应)_

* * *

##### 🛠️ 环境清理

```bash
kubectl delete ingress demo-ingress
kubectl delete deployment app1 app2
kubectl delete svc svc1 svc2
```

```

## 清理之前下载的 controller,反向删除之前配置文件就能清理
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```

#### 知识总结

##### CKA 考点提炼

*   **命令式创建：** 考试中为了节省时间，强烈建议使用快捷命令生成 Ingress 对象： `kubectl create ingress demo-ingress --rule="app1.local/*=svc1:80" --rule="app2.local/*=svc2:80"`
*   **PathType 细节：** `Prefix` 匹配指定前缀的所有子路径（如 `/` 会匹配 `/api` 和 `/login`）。`Exact` 仅进行严格的字符串匹配。
*   **Namespace 限制：** Ingress 资源必须与它要路由的 Service 处于同一个 Namespace 中。跨命名空间的路由需要使用其他高级网关方案。

#### 实验过程问题

1、ADDRESS 那列空代表什么？

执行`kubectl get ingress demo-ingress`命令后，如果发现 ADDRESS 这列空，说明 Ingtress 的规则没有被响应控制器认领。

在 k8s 中，Igress 资源对象本质上只是一段静态配置数据，存在 etcd 数据库里，本身没有任何执行能力。

**“认领”机制：** 当部署了 Ingress Controller（比如 Nginx 网关 Pod）后，这个网关程序会在后台死循环监听（Watch）集群里的 Ingress 资源变化。

*   **认领成功：** 当网关看到一个新建的 Ingress 规则，并且确认 `ingressClassName` 写的是自己的名字（比如 `nginx`），它就会把这些规则写进自己的内存（比如生成 `nginx.conf`）。为了告诉全集群“这个活儿我接了”，网关会主动去修改这个 Ingress 对象的 `Status` 字段，填上自己的 IP。这时候你敲 `kubectl get ingress`，**`ADDRESS` 就会显示出 IP。**

2、多网关情况下流量如何走？

假设集群庞大，为了业务隔离部署了多个网关控制器，如何做到互不干扰？流量走向分为物理链路和逻辑路由两步：

第一步：物理链路入口：

网关本质上是不同的 Pod，它们必定各自拥有一个独立的 Service（NodePort）。

*   Nginx 网关的外部入口可能是：`192.168.44.122:32670`
*   Traefik 网关的外部入口可能是：`192.168.44.122:31111`

**流量去哪，完全取决于用户访问了哪个端口。** 如果你用 `curl` 敲的是 `32670`，这个物理数据包就绝对只会流向 Nginx 的 Pod，Traefik 根本连看都看不到这个请求。

第二步：逻辑路由的匹配（看 IngressClass）

现在，物理流量已经进入了 Nginx 的 Pod。Nginx 怎么决定能不能放行？

*   开发人员提交了一个 Ingress 规则（域名 `app1.local`），里面写了 `ingressClassName: traefik`。
*   Traefik 网关看到了，把它加载到自己的内存里。
*   Nginx 网关看到了，发现名字不对，不加载。

**错误访问场景：** 如果用户访问 `curl -H "Host: app1.local" http://192.168.44.122:32670`（故意把给 Traefik 的域名，打向了 Nginx 的端口）。

*   流量进入 Nginx。
*   Nginx 翻开自己的小本本（路由表），发现自己根本没有收录 `app1.local` 这条规则（因为它只认领 `ingressClassName: nginx` 的规则）。
*   Nginx 当场拒绝，返回 `404 Not Found`。

3、跨节点网络流经过程

假设你当前的架构是：

*   **192.168.44.122 (Node A)**：你发起 `curl` 请求的节点（但上面没有运行网关 Pod）。
*   **192.168.44.123 (Node B)**：Nginx Ingress Controller Pod 实际运行的节点。
*   **NodePort**：32670。

当你的流量打向 `192.168.44.122:32670` 时，完整链路如下：

1.  **第一跳（入站到达 Node A）：** 流量首先到达 Node A 的物理网卡。
2.  **第二跳（kube-proxy 的四层转发与 SNAT）：** Node A 操作系统内核中的 `iptables` 或 `IPVS` 规则（由 kube-proxy 维护）拦截了对 32670 端口的访问。 它发现：这个端口对应的 Nginx Pod 并不在 Node A 上，而是在 Node B (`.123`) 上。 于是，Node A 会做一个**源地址转换（SNAT）**，把数据包的源 IP 改成 Node A 自己，然后通过底层网络（比如你的 Calico）把包转发给 Node B。**（这里产生了额外的网络开销）**
3.  **第三跳（到达网关 Pod 并进行七层解析）：** 数据包终于抵达了 Node B 上的 Nginx Pod。Nginx 拆开 HTTP 报文，看到了 `Host: app1.local`。
4.  **第四跳（网关分发到业务 Pod）：** Nginx 查找自己的路由表，发现应该转给 `svc1`。于是它再次发起连接，将流量转发给运行 `app1` 的业务 Pod（这个 Pod 可能在 Node A、B 或 C 上）。
5.  **返回路径：** 业务 Pod 处理完后，将响应原路返回：App Pod -> Nginx Pod (Node B) -> Node A -> 你的终端。

在普通的测试环境或小型集群中，这种跨节点的局域网转发延迟通常在亚毫秒级，几乎无感。但在高并发的生产环境中，它会带来两个致命问题：

1.  **无谓的网络跳数（Latency）：** 流量多走了一次跨主机网络，增加了延迟，消耗了节点间的带宽。
2.  **丢失真实的客户端 IP：** 因为 Node A 做了 SNAT（源地址转换），当流量到达 Node B 的 Nginx Pod 时，Nginx 看到的源 IP 是 Node A 的内部 IP（`.122`），而不是你真实的外部客户端 IP。这对于需要做 IP 黑白名单过滤或访问统计的业务来说是不可接受的。

在真实的生产环境中，为了压榨性能和保留真实 IP，Kubernetes 工程师通常会采用以下两种优化手段来消灭“多余的跳数”：

优化方案 1：开启 `externalTrafficPolicy: Local` (局部流量策略)

你可以修改 Ingress Controller 的 NodePort Service 配置文件，加入这个参数。

*   **原理解析：** 开启后，kube-proxy 会改变规则。如果流量打到了 Node A（`.122`），但 Node A 上没有 Nginx Pod，**Node A 会直接把这个包丢弃**，拒绝跨节点转发。
*   **效果：** 这样彻底消灭了第二跳，也保留了真实的客户端 IP。
*   **前提条件：** 这种方案必须配合集群外部的物理负载均衡器（比如 F5 或 Nginx/HAProxy）使用。外部负载均衡器会进行健康检查，只把流量发给那些真正运行着 Ingress Pod 的节点（比如直接发给 `.123`）。

优化方案 2：使用 `hostNetwork: true` (主机网络模式)

这是目前国内很多大厂（如裸机环境）最爱用的终极性能方案。 在 Nginx Ingress Controller 的 Deployment 中配置 `hostNetwork: true`。

*   **原理解析：** 此时 Nginx Pod 不再使用 Calico 分配的虚拟 IP，而是**直接绑定物理机（Node B, .123）的网卡和 80/443 端口**。
*   **效果：** 流量直接打到物理机的 80 端口，绕过了所有的 NodePort、Service、kube-proxy 和 iptables 规则。性能几乎等同于在物理机上直接跑一个 Nginx 进程，网络损耗降到最低。