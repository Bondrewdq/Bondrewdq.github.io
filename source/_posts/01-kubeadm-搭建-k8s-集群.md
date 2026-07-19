---
title: 01-Kubeadm 搭建 K8s 集群
tags: []
id: '208'
categories:
  - - k8s 实验
date: 2026-03-02 12:38:56
---

下面是针对 CentOS 环境搭建K8s集群。

**实验拓扑规划：**

*   **Master 节点**: `192.168.44.121` (Hostname: `master`)
*   **Worker 节点 1**: `192.168.44.122` (Hostname: `node1`)
*   **Worker 节点 2**: `192.168.44.12`3 (Hostname: `node2`)

基础网络环境部署参考[使用VMware安装3台虚拟机.pdf](https://www.wqyunpan.com/preview.html?id=151832&fileId=288691)

#### 第一阶段：全局基础环境配置

在 CentOS 系统中，必须先处理 Firewalld 和 SELinux，否则组件之间的通信会被拦截。登录你的终端，分别在三台机器上执行。

**1\. 设置主机名与 hosts 解析**

```bash
## 在 master 节点执行
sudo hostnamectl set-hostname master
## 在 node1 节点执行
sudo hostnamectl set-hostname node1
## 在 node2 节点执行
sudo hostnamectl set-hostname node2

## 在所有节点追加 hosts
cat <<EOF  sudo tee -a /etc/hosts
192.168.44.121 master
192.168.44.122 node1
192.168.44.123 node2
EOF
```

**2\. 关闭防火墙、SELinux 和 Swap**

```bash
## 关闭并禁用防火墙
sudo systemctl stop firewalld
sudo systemctl disable firewalld

## 临时并永久关闭 SELinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

## 临时并永久关闭 Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**3\. 配置内核参数与流量桥接**

```bash
## 加载必要内核模块
cat <<EOF  sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

## 配置 iptables 流量桥接
cat <<EOF  sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

**4\. 安装 Containerd 与配置 Windows 代理** CentOS 推荐通过添加 Docker 官方的源来安装最新版的 `containerd.io`。

```bash
## 安装必要工具并添加源
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

## 安装 containerd
sudo yum install -y containerd.io

## 生成默认配置并开启 SystemdCgroup
sudo mkdir -p /etc/containerd
containerd config default  sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

## 配置通过 Windows 宿主机 Clash 代理拉取 k8s 镜像
sudo mkdir -p /etc/systemd/system/containerd.service.d
cat <<EOF  sudo tee /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://<Windows宿主机IP>:7890/"
Environment="HTTPS_PROXY=http://<Windows宿主机IP>:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.0.0/16,10.244.0.0/16"
EOF

## 启动并设置开机自启
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

**5\. 配置 Kubernetes YUM 源并安装组件** 注意：Kubernetes 官方源已经变更为 `pkgs.k8s.io`。

```bash
cat <<EOF  sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

## 安装核心组件并设置为开机自启（这里暂不启动）
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet
```

* * *

#### 第二阶段：初始化控制平面（📍 仅在 Master 节点执行）

**1\. 执行 kubeadm init** 这里依然使用 Calico 推荐的 `192.168.0.0/16` 作为 Pod 网段。

```bash
[plus@master ~]$ sudo kubeadm init --apiserver-advertise-address=192.168.44.121 --pod-network-cidr=192.168.0.0/16
```

初始化最后输出的 `kubeadm join` 命令及其 Token 和 Hash 复制保存好。

**2\. 配置 kubectl 权限** 为了让能在当前用户下直接管理集群：

```bash
[plus@master ~]$ mkdir -p $HOME/.kube
[plus@master ~]$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[plus@master ~]$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**3\. 安装网络插件 (Calico)**

```bash
[plus@master ~]$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
[plus@master ~]$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

* * *

#### 第三阶段：Worker 节点加入集群（📍 仅在 Node1 和 Node2 节点执行）

分别登录到两个 Worker 节点，执行刚刚保存的加入命令：

```bash
[plus@node1 ~]$ sudo kubeadm join 192.168.44.121:6443 --token <你的token> \
        --discovery-token-ca-cert-hash sha256:<你的hash>
```

* * *

#### 第四阶段：验证集群状态（📍 仅在 Master 节点执行）

回到 Master 节点，观察集群是否就绪：

```bash
[plus@master ~]$ kubectl get nodes -o wide
[plus@master ~]$ kubectl get pods -A
```

#### 相关知识总结

1.  Linux 操作系统与基础环境管理
    
    **关闭 Swap**：K8s 的核心职责是资源调度。如果开启 Swap，当内存不足时系统会将内存数据交换到磁盘，这会导致 Kubelet 无法准确掌握节点的真实内存使用情况，从而做出错误的调度决策。
    
    **SELinux 与 Firewalld**：在 CentOS/RedHat 系中，默认开启的安全策略和防火墙会拦截容器之间以及组件之间的通信端口（如 API Server 的 6443 端口）。理解如何配置或放行这些规则是排障的第一步。
    
2.  Linux 网络栈与流量桥接 (核心考点)
    
    *   **`br_netfilter` 模块**：这个内核模块的作用是让桥接设备（Bridge）上的数据包也能经过 `iptables` 的规则链。
    *   **`net.bridge.bridge-nf-call-iptables=1`**：Kubernetes 的 Service 负载均衡（由 `kube-proxy` 实现）高度依赖 `iptables` 或 `IPVS` 规则。当 Pod 通过 veth pair 将流量发送到宿主机的虚拟网桥时，这个参数确保这些流量会被 `iptables` 拦截并应用 Service 的转发规则。不开启它，集群内部的 DNS 解析和 Service 访问都会失效。
3.  容器运行时接口 (CRI) 与 Cgroup 管理
    
    *   **Containerd**：理解 CRI 的概念，Kubelet 不直接启动容器，而是通过 gRPC 调用 Containerd 来拉取镜像和管理容器生命周期。
        
    *   **`SystemdCgroup = true`**：Linux 使用 Cgroup 来限制进程的资源（CPU、内存）。CentOS 系统的 init 进程是 `systemd`，它本身就是一个 Cgroup 管理器。强制要求 Containerd 也使用 `systemd` 作为 Cgroup 驱动，是为了保证全系统只有一个 Cgroup 管理器，防止资源分配冲突导致节点不稳定。
        
4.  身份验证与 Kubeconfig
    
    *   **Kubeconfig 文件结构**：你需要掌握 `~/.kube/config` 里面包含的三个核心部分：**Clusters**（集群地址和证书）、**Users**（用户凭证/私钥）、**Contexts**（将哪个用户绑定到哪个集群）。
        
    *   **RBAC (基于角色的访问控制)**：`/etc/kubernetes/admin.conf` 中内置了具有 `cluster-admin` 最高权限的证书。在 CKA 考试中，频繁切换 Contexts 是必考操作。
        
5.  集群组件协同 (Control Plane & Worker)
    
    **etcd**：集群的大脑和数据库，存储了所有资源的状态。
    
    **kube-apiserver**：唯一的入口，所有组件（包括 kubelet、用户终端）都只能和 API Server 通信。
    
    **kube-scheduler**：决定新创建的 Pod 应该被分配到 node1 还是 node2（基于资源、污点、亲和性等）。
    
    **kube-controller-manager**：负责维持期望状态（比如确保 Deployment 声明的 3 个副本永远存在，死掉一个就立刻触发新建）。
    
    **kube-proxy**：每个节点上都有，负责维护网络规则，实现 Service 的虚拟 IP 路由。
    
6.  容器网络接口 (CNI)
    
    **Calico/Flannel**：K8s 本身不负责跨主机的 Pod 网络路由，这是由 CNI 插件完成的。你需要理解 `Pod Network CIDR`（如 `192.168.0.0/16`）是如何被分配的，以及 CNI 是如何通过隧道（如 IPIP、VXLAN）或路由规则（如 BGP）让部署在 `node1` 的 Pod 能够直接 ping 通部署在 `node2` 的 Pod 的。
    
    1.  Pod 怎么知道对方在哪里？
        
        在 k8s 中，每个节点（Node）都会被分配一段独立的 Pod 网段，比如 Node1 负责 192.168.1.0/24，Node2 负责 192.168.2.0/24。
        
        Calico 的核心组件 BGP Daemon 会在每个节点运行。比如：
        
        *   **Node2 宣告**：“我有 `192.168.2.0/24` 的路，要找这个网段的包请发给我！”
            
        *   **Node1 学习**：Node1 收到信息后，会在自己的 Linux 内核路由表里加一条线： `192.168.2.0/24 via 192.168.44.122 dev eth0` （去往 Node2 Pod 的包，请从物理网卡发往 Node2 的物理 IP）。
            
    2.  数据包是如何跨越物理网络的？
        
        假设 Node1 上的 Pod A (192.168.1.5) 要 ping Node2 上的 Pod B (192.168.2.10)。
        
        **方式 A：路由模式（直接路由，BGP）**
        
        如果你的 Node1 和 Node2 在同一个二层网络（交换机）下，Calico 默认使用这种方式：
        
        *   **封包**：Pod A 发出原始包（源 IP: 1.5，目标 IP: 2.10）。
        *   **传输**：Node1 看到目标是 2.10，根据路由表，直接把包丢给 Node2 的 MAC 地址。
        *   **拆包**：Node2 收到包，发现目标 IP 2.10 就在自己本地的虚拟网卡上，直接送达。
        *   **特点**：**性能最高**，因为没有额外的拆装包损耗。
        
        **方式 B：隧道模式（IPIP 或 VXLAN）**
        
        如果你的节点跨了子网（路由器不支持 BGP），或者你开启了隧道模式，包就得“坐车”过去。
        
        1.  IPIP 隧道（包中包）
        
        *   **封装**：Node1 发现要跨节点，于是在原始 Pod 包外面再套一层**新的 IP 头部**。
            *   **内层**：源 192.168.1.5 -> 目的 192.168.2.10
            *   **外层**：源 192.168.44.121 -> 目的 92.168.44.122
        *   **传输**：中间的物理交换机只看到这是两个物理机之间的普通通信。
        *   **解封**：Node2 的内核发现是 IPIP 包，拆掉外层，露出里面的 Pod 包，交给 Pod B。
        
        2.  VXLAN 隧道（大二层）
        
        *   **封装**：比 IPIP 更厚。它把整个原始包封装在 **UDP 数据报**里。
        *   **层级**：`物理帧 -> IP头 -> UDP头 -> VXLAN头 -> 原始Pod包`。
        *   **特点**：兼容性极强，即使物理网络只支持简单的 UDP，它也能跑。