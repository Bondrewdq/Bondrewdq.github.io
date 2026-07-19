---
title: 19-集群设置：TLS Ingress 实验
tags: []
id: '432'
categories:
  - - k8s 实验
date: 2026-07-16 13:56:40
---

实验在 Killercoda 进行，自己搭的网络实在是太麻烦了。

##### **步骤 1：安装 NGINX Ingress Controller**

Killercoda 的默认集群没有自带 Ingress 控制器，我们需要手动部署官方的裸机（Bare-metal）版本。

在终端中执行以下命令：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml
```

_提示：安装需要拉取镜像并启动 Pod，你可以运行 `kubectl get pods -n ingress-nginx` 查看状态，等到 `ingress-nginx-controller` 开头的 Pod 变成 `Running` 状态再继续。_

* * *

##### **步骤 2：生成自签名的 TLS 证书**

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=my-secure-app.com/O=my-local-test"
```

**`-x509`**: **最关键的参数**。告诉 OpenSSL 直接输出一个**自签名证书**

**`-nodes`**: 读作 “no DES”。意思是**不对私钥进行加密**。这样生成的 `tls.key` 文件就不需要密码保护。在 Kubernetes Secret 中使用时，必须加这个参数，否则 Pod 启动时会因为无法输入密码解密私钥而报错。

**`-newkey rsa:2048`**: 同时生成一个新的私钥。指定算法为 **RSA**，长度为 **2048** 位

**`-days 365`**: 设置证书的有效期。这里是 1 年。

**`-keyout tls.key`**: 指定生成的**私钥**保存的文件名。

**`-out tls.crt`**: 指定生成的**公钥证书**保存的文件名。

**`-subj "..."`**: 以命令行方式直接提供证书的身份信息，避免进入交互式问答模式。

*   **`/CN` (Common Name)**：域名。比如 `my-secure-app.com`，这要和你的 Ingress 规则里的 `host` 匹配。
*   **`/O` (Organization)**：组织机构名称。

* * *

##### **步骤 3：在 Kubernetes 中创建 TLS Secret**

将生成的证书存入 Secret 中：

```

kubectl create secret tls my-tls-secret \
  --key tls.key \
  --cert tls.crt
```

* * *

##### **步骤 4：部署测试应用 (后端服务)**

部署简单的 NGINX 容器作为后端。可以直接在终端粘贴以下命令，它会使用 `cat` 直接生成并应用配置文件：

```bash
cat <<EOF  kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-app
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello-app
  ports:
  - port: 80
    targetPort: 80
EOF
```

* * *

##### **步骤 5：配置带有 TLS 的 Ingress**

与 Minikube 稍有不同，在标准集群中，显式声明 `ingressClassName: nginx` 是一个好习惯，确保流量被正确接管。

Bash

```bash
cat <<EOF  kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx  # 指定使用我们刚刚安装的 Nginx Ingress
  tls:
  - hosts:
    - my-secure-app.com
    secretName: my-tls-secret
  rules:
  - host: my-secure-app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
EOF
```

* * *

##### **步骤 6：在 Killercoda 中测试 HTTPS**

我们刚才安装的裸机版 Ingress Controller 是通过 **NodePort** 暴露在集群上的。我们需要获取它暴露的 443 (HTTPS) 端口的映射端口。

**1\. 自动提取 HTTPS 的 NodePort 端口号并存入变量：**

```bash
HTTPS_NODEPORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
echo "HTTPS NodePort is: $HTTPS_NODEPORT"
```

**2\. 发起请求测试：** 我们直接在 Killercoda 的终端里，向本机的（`localhost`）这个 NodePort 发起请求，同样强制指定 Host 头：

```bash
curl -kv https://localhost:$HTTPS_NODEPORT -H "Host: my-secure-app.com"
```

\-k： 允许连接到“不安全”的 SSL 站点。

\-v：输出**详细日志**。

\-H：手动注入 HTTP Header 中的 **Host 字段**

**期望结果：** 会在输出信息中看到 `Server certificate: my-secure-app.com` 的 TLS 握手信息，并看到底层 NGINX 应用返回的纯文本内容（如 `Server address` 和 `Server name` 等信息）。这证明在 Killercoda 标准集群上，TLS 终结实验圆满成功！