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
    - [准入控制](#准入控制)
3. [etcd](#etcd)
4. [initializers](#initializers)

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

所以我们的请求已经发送出去了，下一步是什么呢？正如我们提到的，kube-apiserver是客户端和系统组件用于持久化和检索集群状态的主要接口。为了执行其功能，它需要能够验证请求者就是他们所说的人。这个过程称为身份验证。

apiserver如何对请求进行身份验证？当服务器第一次启动时，它将查看安装提供的所有[CLI标志](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)，并组装一个合适的身份验证器列表。让我们举个例子：如果传入了`--client-ca-file`，它将追加x509身份验证器；如果传入了`--token-auth-file`，它将向列表追加令牌身份验证器。每次接收到请求时，都会通过[authenticator链](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54)运行该请求，直到成功:

- [x509 handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60)将验证HTTP请求是否使用有CA根证书签名的TLS密钥编码。
- [bear token handler](ebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38)将验证所提供的令牌（在HTTP`Authorization`头中指定）是否存在于`--token-auth-file`指定的磁盘上的文件中。
- [basicauth handler]将确保HTTP请求的基本身份验证凭证与它自己的本地状态匹配。

如果每个身份验证器都失败，则[请求失败](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71)并返回聚合错误。如果身份验证成功，则从请求中删除授权头，并将[添加用户信息](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75)到其上下文中。这使将来的步骤（如授权和权限控制）能够访问先前建立的用户标识。

### 授权

好的，请求已经发送，kube-apiserver已经成功地验证了我们就是我们所说的人。然而，我们还没有完成。我们可能是我们所说的那样，但是我们有权限执行这个操作吗?毕竟，身份和许可不是一回事。为了我们能继续下去，kube-apiserver需要授权给我们。

kube-apiserver处理授权的方式非常类似于身份验证：基于标志输入，它将组装一个针对每个传入请求运行的授权器链。如果所有授权方都拒绝该请求，则该请求将相应`Forbidden`，[不再继续](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60)。如果单个授权人批准，则请求继续。

与v1.8一起发布的授权者的一些示例包括：

- [webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143)：与集群外HTTP服务交互的。
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223)：执行静态文件中定义的策略。
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43)：执行由管理员作为k8s资源添加的RBAC角色。
- [Node](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67)：确保节点客户端(即kubelet)只能访问驻留在其自身上的资源。

### 准入控制

至此，我们已经通过了kube-apiserver的身份验证和授权。那么剩下还有什么?从kube-apiserver的观点来看，它相信我们是谁，并允许我们继续，但是在Kubernetes看来，系统的其他部分对于什么应该发生，什么不应该发生有着强烈的意见。这是[admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-are-they)介入的地方。

当授权集中于回答用户是否有权限时，准入控制器拦截请求，以确保它符合集群的更广泛期望和规则。它们是将对象持久化到etcd之前的最后一个壁垒，因此它们封装了其余的系统检查，以确保操作不会产生意外或负面结果。

准入控制器的工作方式类似于身份验证器和授权器的工作方式，但有一点不同：与身份验证器和授权器链不同，如果单个准入控制器失败，则整个链将被破坏，请求将失败。

准入控制器设计的真正酷之处在于它关注于促进可扩展性。每个控制器作为单独的插件存储在[`plugin/pkg/admission`目录](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission)中，并满足一个小的接口，然后，每一个插件都被编译进kubernetes主课本身。

准入控制器通常分为资源管理、安全性、默认值和引用一致性。下面是一些只负责资源管理的准入控制器的例子:

- `InitialResources`根据过去的使用情况为容器的资源设置默认资源限制。
- `LimitRanger`为容器requests和limits设置默认值，或保证某些资源上限(不超过2GB内存，默认为512MB)。
- `ResourceQuota`计算和拒绝名称空间中的许多对象(pods、rc)或总消耗资源(cpu、内存、磁盘)

## etcd

至此，Kubernetes已经完全审查了传入的请求，并允许它继续运行并获得成功。下一步，kube-apiserver反序列化HTTP请求，从它们构造运行时对象（有点像kubectl生成器的逆过程），并将它们持久化到数据存储中。让我们把它分解一下。

kube-apiserver如何知道在接受我们的请求时应该做什么？在任何请求被服务之前都有一系列非常复杂的步骤。让我们从头开始，从二进制文件第一次运行开始：

1. 当`kube-apiserver`二进制文件运行时，它[创建一个服务器链](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L119)，这允许apiserver聚合。这实际上是一种支持多个apiserver的方法（我们不必关心这个）。
2. 当这种情况发生时，将创建作为默认实现的[通用apiserver](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149)。
3. 生成的OpenAPI模式填充[apiserver的配置](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149)。
4. 然后，kube-apiserver遍历模式中指定的所有API组，并为每个服务配置一个[存储提供器](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410)作为通用存储抽象。kube-apiserver会在访问或改变资源状态时通知它。
5. 对于每个API组，它还遍历每个组版本，并为每个HTTP路由[安装REST映射](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92)。这允许kube-apiserver映射请求，并能够在找到匹配项后将其委托给正确的逻辑。
6. 对于我们的特定用例，注册了一个[POST handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710)，然后它将委托给一个[create resource handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)。

至此，kube-apiserver完全知道存在哪些路由，并具有一个内部映射，用来判断当请求匹配时调用哪些处理程序和存储提供程序。现在让我们想象我们的HTTP请求已经流入了：

1. 如果处理程序链能够将请求匹配到一个集合模式（即我们注册的路由），那么它将分配为该路由注册的[专用handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143)。否则，它将退回到基于路径的处理程序（这是调用`/apis`时发生的情况）。如果没有为该路径注册处理程序，则调用[not found handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254)，结果是404。
2. 幸运的是，我们有一个名为[createHandler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)的注册路由！它是做什么的？它将首先解码HTTP请求并执行基本验证，例如确保它们提供的JSON与我们对版本化API资源的期望相关。
3. 将进行[审核和最终准入](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104)。
4. 通过[委托给存储提供程序](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327)，资源将被[保存到etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111)。通常etcd键的形式是`<namespace>/<name>`，但这是可配置的。
5. 任何创建错误都会被捕获，最后，存储提供程序执行`get`调用，以确保实际创建了对象。然后，如果需要额外的终结，它将调用任何创建后处理程序和装饰器。
6. [构建HTTP响应](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142)并发回。

## initializers

在将对象持久化到数据存储之后，apiserver不会使其完全可见，也不会对其进行调度，直到运行了一系列的[初始化器](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#initializers)。初始化器是与资源类型相关联的控制器，在资源对外可用之前对其执行逻辑。如果资源类型没有注册任何初始化器，则跳过此初始化步骤并立即使资源可见。

正如许多[优秀的博客文章](https://ahmet.im/blog/initializers/)所指出的，这是一个强大的特性，因为它允许我们执行一般的引导操作。例子可以是：

- 将代理sidecar容器注入到公开端口80的Pod中，或提供特定的注释。
- 将带有测试证书的存储卷注入到特定名称空间中的所有Pods中。
- 如果Secret小于20个字符(例如密码)，则阻止创建它。

`initializerConfiguration`对象允许您声明应该为某些资源类型运行哪些初始化程序。假设我们希望在每次创建Pod时都运行自定义初始化器，我们会这样做：

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```

创建此配置后，它将向每个Pod的`metadata.initializers.pending`字段增加`custom-pod-initializer`。初始化器控制器将已经部署，并将定期扫描新的Pods。当初始化器在Pod的pending字段中检测到带有其名称的项时，它将执行其逻辑。在完成它的流程之后，它将从挂起列表中删除它的名称。只有名称在列表中名列第一的初始化器才能对资源进行操作。当所有初始化器完成且pending字段为空时，将认为对象已初始化。

眼尖的你可能已经发现了一个潜在的问题。如果kube-apiserver无法使这些资源可见，那么用户域控制器如何处理这些资源呢？为了解决这个问题，kube-apiserver公开了一个`?includeUninitialized`查询参数，该参数返回所有对象，甚至是统一的对象。
