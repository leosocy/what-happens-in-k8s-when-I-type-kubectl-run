# 当我在终端输入`kubectl run`时都发生了什么

假设我想将nginx部署到Kubernetes集群中。我可能会在终端机里输入这样一行命令:

```shell
kubectl run --image=nginx --replicas=3
```

然后按下回车键。几秒钟之后，我应该看到三个nginx pods分布在我的工作节点上。它像魔术一样神奇，太棒了！但在这条命令背后到底发生了什么?

Kubernetes最棒的一点是，它通过用户友好的api处理跨基础设施的部署。复杂性隐藏在简单的抽象中。但是，为了充分理解它为我们提供的价值，理解它的内在也时很有用处滴。本指南将引导你理解从客户端到kubelet的请求完整生命周期下发生了什么，并在必要时链接到源代码，以说明正在发生的事情。

## 内容摘要

1. [kubectl](#kubectl)
    - [验证，生成运行时对象](#验证，生成运行时对象)
    - [api组和版本协商](#api组和版本协商)
    - [客户端身份验证](#客户端身份验证)
2. [kube-apiserver](#kube-apiserver)
    - [认证](#认证)
    - [授权](#授权)
    - [权限控制](#权限控制)

## kubectl

### 验证，生成运行时对象

好，我们开始吧。我们刚在终端机按回车键。现在该做什么?

kubectl要做的第一件事是执行一些客户端验证。这确保总是失败的请求(例如创建不受支持的资源或使用[不正确的镜像名称](https://github.com/kubernetes/kubernetes/blob/9a480667493f6275c22cc9cd0f69fb0c75ef3579/pkg/kubectl/cmd/run.go#L251))将快速失败，不会发送到kube-apiserver。这通过减少不必要的负载来提高系统性能。

验证之后，kubectl开始组装它将发送给kube-apiserver的HTTP请求。所有访问或更改Kubernetes系统中的状态的尝试都要通过API服务器，而API服务器又与etcd通信。kubectl客户机也不例外。为了构造HTTP请求，kubectl使用一种称为[generators](https://kubernetes.io/docs/reference/kubectl/conventions/#generators)的东西，生成器是负责序列化的抽象。

可能不太明显的是，你实际上可以使用kubectl运行来指定多种资源类型，而不仅仅是部署。为了实现这一点，如果没有使用`--generator`标志显式指定生成器名称，那么kubectl将推断资源类型。

例如，具有`--restart-policy=Always`的资源被认为是deployment，而具有`--restart-policy=Never`的资源被认为是pod。kubectl还将确定是否需要触发其他操作，例如记录命令(用于回滚或审计)，或者该命令是否只是一个预演(由`--dry-run`标志指示)。

在kubectl意识到我们希望创建一个部署之后，它将使用`DeploymentV1Beta1`生成器从我们提供的参数生成一个[运行时对象](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/run.go#L59)。“运行时对象”是资源的通用术语。

### api组和版本协商

在继续之前，值得指出的是，Kubernetes使用了一个*版本化*的API，它被划分为“API组”。API组旨在对类似的资源进行分类，以便更容易对其进行推理。它还提供了单一单片API的更好的替代方案。部署的API组名为apps，其最新版本为v1。这就是为什么需要在deployment配置文件的开始声明`apiVersion: apps/v1`。

不管怎样，在kubectl生成运行时对象之后，它开始为其寻找合适的API组和版本，然后组装一个版本化的客户端，该客户端知道资源的各种REST语义。这个发现阶段称为版本协商，涉及到kubectl扫描远程API上的`/apis`路径以检索所有可能的API组。由于kube-apiserver在这个路径上公开了它的OpenAPI格式的文档，所以客户端很容易执行自己的发现。

为了提高性能，kubectl还将OpenAPI模式[缓存](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/util/factory_client_access.go#L117)到`~/.kube/cache/discovery`。

最后一步是实际发送HTTP请求。这样做并返回一个成功的响应之后，kubectl将根据所需的输出格式打印一条成功消息。

### 客户端身份验证

在前面的步骤中，我们没有提到的一件事是客户端身份验证(在发送HTTP请求之前进行处理)，所以现在让我们来看一下。

为了成功地发送请求，kubectl需要能够进行身份验证。用户凭证几乎总是存储在磁盘上的`kubeconfig`文件中，但是该文件可以存储在不同的位置。为了找到它，kubectl做了以下工作:

- 如果提供了`--kubeconfig`参数，使用它。
- 如果定义了`$KUBECONFIG`环境变量，使用它。
- 否则在[推荐的目录](https://github.com/kubernetes/client-go/blob/master/tools/clientcmd/loader.go#L52)中查找，并使用找到的第一个文件。

在解析文件之后，它将确定当前要使用的上下文、当前要指向的集群以及与当前用户关联的任何身份验证信息。如果用户提供了特定于标志的值(如`--username`)，则这些值优先，并将覆盖kubeconfig中指定的值。一旦获得这些信息，kubectl就会填充客户端的配置，以便能够适当地修饰HTTP请求:

- x509使用[tls.TLSConfig](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89)发送，这也包括了CA证书。
- bear tokens通过[`Authorization`HTTP头](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314)发送。
- username和password通过[HTTP基本身份验证](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223)发送。
- OpenID身份验证过程由用户事先手动处理，生成一个令牌，该令牌像bear tokens一样发送

## kube-apiserver

### 认证

### 授权

### 权限控制