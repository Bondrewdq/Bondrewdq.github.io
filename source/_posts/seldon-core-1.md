---
title: Seldon Core 1
tags: []
id: '440'
categories:
  - - seldon core
date: 2026-07-17 15:58:34
---

Seldon Core一个运行在 Kubernetes 上的机器学习模型服务平台，用来把训练好的模型部署成可访问、可扩缩、可观测的在线推理服务。这里学习的是1版本。

它解决的不是“怎么训练模型”，而是：

*   模型训练好了，怎么部署上线？
*   怎么暴露 REST / gRPC 接口？
*   怎么做版本切换、流量分配？
*   怎么同时管理大量模型？
*   怎么监控延迟、错误率和资源使用？
*   怎么把多个模型、预处理和后处理串成推理流水线？

> 当前官方重点已经转向 **Seldon Core 2**。Core 2 被定位为 Kubernetes 上的 MLOps/LLMOps 框架，可以管理单模型、多个模型以及模块化推理应用。

### 架构

Seldon Core 1 在K8s 上增加了一个更高层的抽象：SeldonDeployment。

这里描述的是模型推理服务，Seldon Operator 再将它转换成 k8s 能运行的资源。

#### 认识 SeldonDeployment

官方将 `SeldonDeployment` 定义为 Kubernetes 自定义资源，用户通过它描述模型组件和推理图。

一个简单示例如下：

```yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: seldon
spec:
  name: iris
  predictors:
    - name: default
      replicas: 1

      graph:
        name: classifier
        type: MODEL
        implementation: SKLEARN_SERVER
        modelUri: gs://seldon-models/sklearn/iris

      componentSpecs:
        - spec:
            containers:
              - name: classifier
                resources:
                  requests:
                    cpu: 100m
                    memory: 256Mi
                  limits:
                    cpu: 1
                    memory: 1Gi
```

这一份 YAML 里混合了两类信息：

```

模型语义
+
Kubernetes 运行配置
```

1.  `spec.predictors`
    
    ```
    predictors:
        - name: default
    ```
    
    一个 predictors 可以理解为：一套可以独立接收推理请求的模型服务配置。
    
    其中可以包含：
    
    ```
    模型；
    输入转换器；
    输出转换器；
    路由器；
    组合器；
    副本数量；
    Pod 配置。
    ```
    
2.  `replicas`
    
    ```
    repicas: 1
    ```
    
    描述该 predictor 的副本数量。但实际生成资源时，可能根据图结构和组件配置创建一个或多个 Deployment，因此不要机械认为一个 predictor 永远只对应一个 Deployment。
    
3.  `graph`
    
    ```
    graph:
      name: classifier
      type: MODEL
      implementation: SKLEARN_SERVER
      modelUri: gs://seldon-models/sklearn/iris
    ```
    
    是 Seldon 相比普通 Deployment 最核心的能力之一，描述的是推理组件之间的调用关系。
    
    只有一个模型时：
    
    ```
    request
       ↓
    classifier
       ↓
    response
    ```
    
    稍微复杂一点：
    
    ```
    request
       ↓
    input-transformer
       ↓
    classifier
       ↓
    output-transformer
       ↓
    response
    ```
    
    还可以存在：
    
    ```
    request
       ↓
    router
     ┌─┴─────────┐
     ↓           ↓
    model-a    model-b
     └────┬──────┘
          ↓
       combiner
          ↓
      response
    ```
    
    Seldon Core 1 官方文档说明，推理图可以由模型以及 router、combiner、输入和输出 transformer 等组件构成。
    
    普通 Kubernetes Deployment 只知道：
    
    ```
    运行哪些容器
    ```
    
    SeldonDeployment 还知道：
    
    ```
    这些模型组件之间应该如何调用
    ```
    
    `graph` 是根节点，`children` 表示连接的下一层节点，多个 `-` 表示多个并列节点，`implementation` 选择预打包服务器，`modelUri` 指向模型目录。
    
4.  `implementation`
    
    ```
    implementation: SKLEARN_SERVER
    ```
    
    意思是：
    
    > 使用 Seldon 已经准备好的 sklearn 推理服务器运行模型。
    
    Seldon Core 1 提供了多种预打包推理服务器，例如：
    
    *   SKLearn Server；
    *   XGBoost Server；
    *   TensorFlow Serving；
    *   MLflow Server；
    *   自定义服务器。
    
    因此你不一定要自己完整实现：
    
    ```
    HTTP Server
    请求反序列化
    模型加载
    predict 调用
    响应序列化
    ```
    
    这是 Seldon 比“直接写 Deployment”多做的一层标准化。
    
    > 什么是预打包服务器？
    > 
    > **预**：提前准备好
    > 
    > **打包**：把代码、依赖库、运行环境放进一个容器镜像里
    > 
    > “服务器”则表示它不是普通脚本，而是一个会持续运行、接收网络请求并返回预测结果的程序。
    > 
    > 所以预打包服务器指：提前打包好的，用于运行模型的服务器程序。之所以叫这个名字，核心原因就是：服务器代码和运行环境已经由 Seldon 提前封装好了，你不需要自己再写和配置。
    
5.  `modelUri`
    
    ```
    modelUri: gs://seldon-models/sklearn/iris
    ```
    
    描述模型文件的位置，而不是容器镜像的位置。
    
    Seldon 可以使用预打包模型服务器，然后根据 `modelUri` 加载模型。官方 SKLearn 示例就是通过 `implementation` 和 `modelUri` 描述模型服务。
    
6.  `conponentSpec`
    
    ```
    componentSpecs:
      - spec:
          containers:
            - name: classifier
              resources:
                requests:
                  cpu: 100m
                  memory: 256Mi
    ```
    
    非常接近 Kubernetes 的：
    
    ```
    PodTemplateSpec
    ```
    
    它用于描述：
    
    *   容器；
    *   环境变量；
    *   CPU、内存；
    *   Volume；
    *   VolumeMount；
    *   安全上下文；
    *   节点选择；
    *   GPU 资源；
    *   其他 Pod 配置。
    
    因此可以这样理解：
    
    ```
    graph
    ```
    
    描述模型推理逻辑。
    
    ```
    componentSpecs
    ```
    
    描述模型组件如何在 Kubernetes 中运行。
    

#### Sldon Operator

核心循环抽象：

```

func Reconcile(request Request) {
    // 1. 获取 SeldonDeployment
    sdep := getSeldonDeployment(request)

    // 2. 根据 spec 计算期望资源
    desiredDeployments := buildDeployments(sdep)
    desiredServices := buildServices(sdep)

    // 3. 查询当前资源
    currentDeployments := getDeployments(sdep)
    currentServices := getServices(sdep)

    // 4. 对比期望状态与当前状态
    diff := compare(
        desiredDeployments,
        currentDeployments,
    )

    // 5. 创建、更新或删除资源
    apply(diff)

    // 6. 更新状态
    updateStatus(sdep)
}
```

#### 与直接写 Deployment 相比

1.  模型语义 Deployment 只知道运行镜像，SeldonDeployment 能表达：
    
    ```
    这是 MODEL
    这是 ROUTER
    这是 TRANSFORMER
    这是 COMBINER
    ```
    
2.  推理图
    
    Deployment 本身不能直接表达：
    
    ```
    请求 → 预处理 → 模型 → 后处理
    ```
    
    SeldonDeployment 可以通过 `graph` 描述这种关系。
    
3.  标准推理服务器
    
    普通 Deployment 通常需要你自己准备完整模型服务镜像。
    
    Seldon 可以使用预打包服务器运行 sklearn、XGBoost、MLflow 等模型。