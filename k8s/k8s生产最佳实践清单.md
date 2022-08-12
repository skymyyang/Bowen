## kubernetes生产环境最佳实践

本文仅提供在kubernetes上部署安全、可扩展和弹性服务的可行性最佳实践。仅供参考。

精选的最佳实践清单，旨在帮助您发布到生产环境。



### 应用开发

#### 健康检查

- **容器要有就绪探针**

  注意：readiness probe（就绪探针） 和 liveness probe（存活探针） 没有默认值。

  如果您没有设置就绪探测，kubelet 会假定应用程序已准备好在容器启动后立即接收流量。

  如果容器需要 2 分钟才能启动，那么在这 2 分钟内对它的所有请求都将失败。

- **发生不可恢复错误（致命错误）时让容器崩溃**

  如果应用程序遇到不可恢复的错误，[您应该让它崩溃](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/#letitcrash)。

  此类不可恢复错误的示例如下：

  - 未捕获的异常
  - 代码中的错字（对于动态语言）
  - 无法加载标头或依赖项

  请注意，您不应发出失败的 Liveness 探测信号。

  相反，您应该立即退出进程并让 kubelet 重新启动容器。

- **配置被动liveness探测**

  Liveness 探针旨在在容器卡住时重新启动容器。

  考虑以下场景：如果您的应用程序正在处理无限循环，则无法退出或寻求帮助。

  当进程消耗 100% CPU 时，它没有时间回复（其他）Readiness 探测检查，最终会从 Service 中删除。

  但是，Pod 仍然注册为当前 Deployment 的活动副本。

  如果您没有 Liveness 探针，它会保持*运行*但与服务分离。

  换句话说，该进程不仅不服务任何请求，而且还在消耗资源。

  *你该怎么办？*

  1. 从您的应用程序公开端点
  2. 端点始终回复成功响应
  3. 使用 Liveness 探针中的端点

  请注意，您不应使用 Liveness 探针来处理应用程序中的致命错误并请求 Kubernetes 重新启动应用程序。

  相反，您应该让应用程序崩溃。

  只有在进程没有响应的情况下，才应将 Liveness 探针用作恢复机制。

- readiness probe（就绪探针） 和 liveness probe（存活探针）的值要确保不相同

  如果Liveness和Readiness值相同时，如果应用程序发出未准备好或未上线的信号时，kubelet会将容器从服务中分离并删除，可能会导致连接丢失。因为容器没有足够的时间来处理传入的连接。

#### 应用程序独立部署

**readiness probe（就绪探针）是独立的**

就绪探测不包括对服务的依赖，例如：

- databases
- database migrations
- APIs
- 第三方服务

参考链接：`https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/#shootingyourselfinthefootwithreadinessprobes`

**应用程序的重连机制**

当应用程序启动时，它不应该因为数据库等依赖项尚未准备好而崩溃。

相反，应用程序应该不断重试连接到数据库，直到成功。

Kubernetes 期望应用程序组件可以以任何顺序启动。

当您确保您的应用程序可以重新连接到依赖项（例如数据库）时，您就知道您可以提供更强大和更有弹性的服务。

#### 优雅关闭应用程序

**应用程序不会再接收到SIGTERM后立即关闭，等待一段时间后，优雅终止连接**

pod即使在收到终止信号后，可能也需要继续接受连接，直到所有的kube-proxy完成对iptables规则或ipvs规则的更新。

可能ingress-controller以及Loadblance也会将连接转发到POD。

为确保所有客户端都不会遇到断开连接，您必须等到所有客户端都以某种方式通知您他们将不再将连接转发到 pod。

但是显然，这是不可能的，因为所有这些组件都分布在许多不同的计算机上。如果有一个没有写响应，都会导致应用无法关闭。

一般情况下我们会配置一个预停止的hook：

```yaml
lifecycle:
      preStop:
        exec:
          command:
          - sh
          - c
          - "sleep 5"
```

**将SIGTERM信号转发给进程**

当pod即将终止时，可以通过在应用程序中捕获SIGTERM信号。可以使用tini。tini项目地址：`https://github.com/krallin/tini`

**关闭所有的空闲的kaap-alive套接字**

如果调用应用程序没有关闭TCP连接；当pod被删除时；理想情况下，请求应该发送到另一个 Pod；但是，调用应用程序与即将终止的 Pod 建立了长期连接，并将继续使用它；不应该突然终止长期连接；您应该在关闭应用程序之前终止它们。



#### 容错机制

**为部署的POD运行多个副本**

永远不要单独运行单个 Pod。

而是考虑将 Pod 部署为 Deployment、DaemonSet、ReplicaSet 或 StatefulSet 的一部分。

[运行多个 Pod 实例可确保删除单个 Pod 不会导致停机](https://cloudmark.github.io/Node-Management-In-GKE/#replicas)。

**避免将 Pod 放置到单个节点中**

考虑以下场景：单个集群节点上有 11 个副本。

如果节点不可用，则 11 个副本将丢失，并且您有停机时间。

[您应该将反关联规则应用于您的部署，以便 Pod 分布在集群的所有节点中](https://cloudmark.github.io/Node-Management-In-GKE/#pod-anti-affinity-rules)。

pod间[亲和性和反亲和性](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity)文档描述了如何将 Pod 更改为位于（或不在）同一节点中。

设置 Pod 中断预算

**Set Pod disruption budgets**

翻译过来就是配置POD干扰预算；通过设置应用 Pod 处于正常状态的最低个数或最低百分比，这样可以保证在主动销毁 Pod 的时候，不会销毁太多的 Pod 导致业务异常中断，从而提高业务的可用性。

参考链接：`https://zhuanlan.zhihu.com/p/367827614`

[官方文档是了解Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)`https://kubernetes.io/docs/concepts/workloads/pods/disruptions/`

#### 资源利用

**为所有的容器设置内存限制和请求**

资源限制用于限制您的容器可以使用多少 CPU 和内存，并使用`containerSpec` 字段进行配置。

调度程序使用这些作为指标之一来决定哪个节点最适合当前 Pod。

没有做资源限制的容器调度优先级为0.即当发生OOM时，会先停掉此类容器。

由于 CPU 是一种可压缩资源，因此如果您的容器超出限制，则进程会受到限制。

请注意，如果您不确定*正确*的CPU 或内存限制应该是多少，您可以在 Kubernetes 中使用[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)并打开推荐模式。自动缩放器会分析您的应用并为其推荐限制。

**将 CPU 请求设置为 1 个 CPU 或以下**

除非您有计算密集型作业，[否则建议将请求设置为 1 CPU 或以下](https://www.youtube.com/watch?v=xjpHggHKm78)。

**禁用CPU限制**

如果您不确定应用程序的最佳设置是什么，最好不要设置 CPU 限制。

如果您想了解更多信息，[本文将深入探讨 CPU 请求和限制](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)。`https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b`

**为namesapce（名称空间）设置LimitRange**

如果您认为您可能忘记设置内存和 CPU 限制，您应该考虑使用 LimitRange 对象来定义部署在当前命名空间中的容器的标准大小。

[LimitRange 的官方文档](https://kubernetes.io/docs/concepts/policy/limit-range/)`https://kubernetes.io/docs/concepts/policy/limit-range/`

**为Pod设置适当的Qos（服务质量）**

当一个节点资源使用率过高时，kubernetes会尝试驱驱逐该节点中的一些pod。

kubernetes会根据定义的优先级以及Qos对pod进行排名和驱逐。

关于如何配置Qos[可参考官方文档](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) :`https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/`

#### 资源标记

**定义技术相关标签**

下面是[官方推荐](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)的相关技术的标签：

官方文档：`https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/`

- `name`, 应用程序的名称，例如“用户 API”
- `instance`，标识应用程序实例的唯一名称（您可以使用容器图像标签）
- `version`，应用程序的当前版本（增量计数器）
- `component`，架构内的组件，例如“API”或“数据库”
- `part-of`, 一个更高级别的应用程序的名称，这是“支付网关”的一部分
- `managed-by`，用于管理应用程序操作的工具，例如“kubectl”或“Helm”

示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    app.kubernetes.io/name: user-api
    app.kubernetes.io/instance: user-api-5fa65d2
    app.kubernetes.io/version: "42"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: payment-gateway
    app.kubernetes.io/managed-by: kubectl
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: user-api
        app.kubernetes.io/instance: user-api-5fa65d2
        app.kubernetes.io/version: "42"
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: payment-gateway
    spec:
      containers:
      - name: app
        image: myapp
```

**定义业务标签**

您可以使用以下标签标记您的 Pod：

- `owner`，用于标识谁负责该资源
- `project`，用于确定资源所属的项目
- `business-unit`，用于标识与资源关联的成本中心或业务单位；通常用于成本分配和跟踪。



示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    owner: payment-team
    project: fraud-detection
    business-unit: "80432"
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        owner: payment-team
        project: fraud-detection
        business-unit: "80432"
    spec:
      containers:
      - name: app
        image: myapp
```

**定义安全标签**

您可以使用以下标签标记您的 Pod：

- `confidentiality`，资源支持的特定数据机密级别的标识符
- `compliance`，旨在遵守特定合规性要求的工作负载标识符

示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    confidentiality: official
    compliance: pci
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        confidentiality: official
        compliance: pci
    spec:
      containers:
      - name: app
        image: myapp
```

#### 日志记录

**应用程序日志输出到`stdout`和`stderr`**

有两种日志记录策略：*passive（被动）* 和 *active（主动）*

使用被动策略的应用程序将日志消息记录到标准输出。

此最佳实践是[十二因素应用程序的](https://12factor.net/logs)一部分。

在主动日志记录中，应用程序与中间聚合器建立网络连接，将数据发送到第三方日志记录服务，或直接写入数据库或索引。

主动日志记录被认为是一种反模式，应该避免。

**避免使用sidecar进行日志记录（如果可以的话）**

如果您希望[将日志转换应用于具有非标准日志事件模型的应用程序](https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure)，您可能需要使用边车容器。

使用 sidecar 容器，您可以在将日志条目发送到其他地方之前对其进行规范化。

例如，您可能希望在将 Apache 日志传送到日志基础设施之前将其转换为 Logstash JSON 格式。

但是，如果您可以控制应用程序，则可以首先输出正确的格式。

您可以节省为集群中的每个 Pod 运行一个额外的容器。

#### 缩放

**容器不会在其本地文件系统中存储任何状态**

容器有一个本地文件系统，你可能会想用它来持久化数据。

但是，将持久数据存储在容器的本地文件系统中会阻止包含的 Pod 水平扩展（即，通过添加或删除 Pod 的副本）。

这是因为，通过使用本地文件系统，每个容器都维护自己的“状态”，这意味着 Pod 副本的状态可能会随着时间的推移而发散。这会导致从用户的角度来看不一致的行为（例如，当请求命中一个 Pod 时，一条特定的用户信息可用，但当请求命中另一个 Pod 时则不可用）。

相反，任何持久性信息都应该保存在 Pod 之外的中心位置。例如，在集群中的一个 PersistentVolume 中，甚至在集群外的一些存储服务中更好。

**将 Horizontal Pod Autoscaler 用于具有可变使用模式的应用程序**

Horizontal [Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)是一个内置的 Kubernetes 功能，可以监控您的应用程序并根据当前使用情况自动添加或删除 Pod 副本。

配置 HPA 可让您的应用在任何流量条件下（包括意外高峰）保持可用和响应。

要将 HPA 配置为自动缩放您的应用程序，您必须创建一个[HorizontalPodAutoscaler](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#horizontalpodautoscaler-v1-autoscaling)资源，该资源定义了要为您的应用程序监控的指标。

HPA 可以监控内置资源指标（Pod 的 CPU 和内存使用情况）或自定义指标。在自定义指标的情况下，您还负责收集和公开这些指标，例如，您可以使用[Prometheus](https://prometheus.io/)和[Prometheus Adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)来做到这一点。

**请勿在 Vertical Pod Autoscaler 仍处于测试阶段时使用它**

与[Horizontal Pod Autoscaler (HPA)类似， ](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)[Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)也存在。

VPA 可以自动适应你 Pod 的资源请求和限制，这样当一个 Pod 需要更多资源时，它可以得到它们（增加/减少单个 Pod 的资源称为*垂直扩展*，而不是*水平扩展*，这意味着增加/减少 Pod 的副本数）。

这对于扩展无法水平扩展的应用程序很有用。

然而，VPA 目前处于 beta 阶段，它有[一些已知的限制](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#limitations-of-beta-version)（例如，通过改变 Pod 的资源需求来扩展 Pod，需要杀死并重新启动 Pod）。

鉴于这些限制，以及 Kubernetes 上的大多数应用程序无论如何都可以水平扩展的事实，建议不要在生产中使用 VPA（至少在有稳定版本之前）。

**如果您有高度变化的工作负载，请使用 Cluster Autoscaler**

[Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)是另一种类型的“autoscaler”（除了Horizo [ntal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)和[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)）。

Cluster Autoscaler 可以通过添加或删除工作节点来自动扩展集群的大小。

当 Pod 由于现有工作节点上的资源不足而无法调度时，就会发生扩展操作。在这种情况下，Cluster Autoscaler 会创建一个新的工作节点，以便 Pod 可以被调度。同样，当现有工作节点的利用率较低时，集群自动缩放器可以通过从一个工作节点中逐出所有工作负载并将其移除来缩减规模。

使用 Cluster Autoscaler 对于高度可变的工作负载是有意义的，例如，当 Pod 的数量可能在短时间内增加，然后又回到之前的值时。在这种情况下，Cluster Autoscaler 允许您通过过度配置工作节点来满足需求高峰，而不会浪费资源。

但是，如果您的工作负载变化不大，则可能不值得设置 Cluster Autoscaler，因为它可能永远不会被触发。如果您的工作负载增长缓慢且单调，那么监控现有工作节点的利用率并在达到临界值时手动添加一个额外的工作节点可能就足够了。

#### 配置和Secret

**外部化所有配置**

配置应在应用程序代码之外维护。

这有几个好处。首先，更改配置不需要重新编译应用程序。其次，可以在应用程序运行时更新配置。第三，相同的代码可以在不同的环境中使用。

在 Kubernetes 中，可以将配置保存在 ConfigMaps 中，然后可以将其挂载到容器中，因为卷作为环境变量传入。

在 ConfigMaps 中仅保存非敏感配置。对于敏感信息（例如凭证），请使用 Secret 资源。

**将 Secret 挂载为卷，而不是环境变量**

Secret 资源的内容应该作为卷挂载到容器中，而不是作为环境变量传入。

这是为了防止秘密值出现在用于启动容器的命令中，这可能会被不应访问秘密值的个人查看到。

### Governance(治理)

#### 命名空间限制

**名称空间配置LimitRange**

没有限制的容器可能会导致与其他容器的资源争用和计算资源的未优化消耗。

Kubernetes 有两个限制资源使用的特性：ResourceQuota 和 LimitRange。

使用 LimitRange 对象，您可以定义资源请求的默认值和命名空间内单个容器的限制。

在该命名空间内创建的任何容器，没有明确指定请求和限制值，都会被分配默认值。

[资源配额](https://kubernetes.io/docs/concepts/policy/resource-quotas/)，您应该查看官方文档。`https://kubernetes.io/docs/concepts/policy/resource-quotas/`

**名称空间配置ResourceQuota**

使用 ResourceQuotas，您可以限制命名空间内所有容器的总资源消耗。

为命名空间定义资源配额会限制属于该命名空间的所有容器可以使用的 CPU、内存或存储资源的总量。

您还可以为其他 Kubernetes 对象设置配额，例如当前命名空间中的 Pod 数量。

如果您认为有人可以利用您的集群并创建 20000 个 ConfigMap，那么使用 LimitRange 可以防止这种情况发生。

#### Pod安全策略

**启用Pod安全策略**

可以使用 Kubernetes Pod 安全策略来限制：

- 访问主机进程或网络命名空间
- 运行特权容器
- 容器运行的用户
- 访问主机文件系统
- Linux 功能、Seccomp 或 SELinux 配置文件

参考[Kubernetes Pod安全策略最佳实战](https://www.mend.io/resources/blog/kubernetes-pod-security-policy/) `https://www.mend.io/resources/blog/kubernetes-pod-security-policy/`

**禁用特权容器**

在 Pod 中，容器可以在“特权”模式下运行，并且几乎可以不受限制地访问主机系统上的资源。

虽然在某些特定用例中需要此级别的访问权限，但总的来说，让您的容器执行此操作会带来安全风险。

特权 Pod 的有效用例包括使用节点上的硬件，例如 GPU。

您可以[从本文中了解有关安全上下文和权限容器的更多信息](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)  ：`https://kubernetes.io/docs/tasks/configure-pod-container/security-context/`

**在容器中使用只读文件系统**

在容器中运行只读文件系统会强制容器不可变。

这不仅可以减轻一些旧的（和有风险的）做法，例如热补丁，还可以帮助您防止恶意进程在容器内存储或操作数据的风险。

使用只读文件系统运行容器可能听起来很简单，但它可能会带来一些复杂性。

*如果您需要在临时文件夹中写入日志或存储文件怎么办？*

[您可以在本文中了解有关在生产环境中安全运行容器](https://medium.com/@axbaretto/running-docker-containers-securely-in-production-98b8104ef68)的权衡取舍： `https://medium.com/@axbaretto/running-docker-containers-securely-in-production-98b8104ef68`

**防止容器以root身份运行**

在容器中运行的进程与主机上的任何其他进程没有什么不同，只是它有一小段元数据声明它在容器中。

因此，容器中的 root 与主机上的 root (uid 0) 相同。

如果用户设法突破在容器中以 root 身份运行的应用程序，他们可能能够获得对具有相同 root 用户的主机的访问权限。

将容器配置为使用非特权用户，是防止提权攻击的最佳方法。

如果您想了解更多信息，以下[文章提供了一些详细说明示例，说明当您以 root 身份运行容器时会发生什么](https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b) ：`ttps://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b`

**限制一些Linux功能**

Linux 功能使进程能够执行默认情况下只有 root 用户才能执行的许多特权操作。

例如，`CAP_CHOWN`允许进程“对文件 UID 和 GID 进行任意更改”。

即使您的进程不以 身份运行`root`，进程也有可能通过提升权限来使用这些类似 root 的功能。

换句话说，如果您不想受到损害，您应该只启用您需要的功能。

*但是应该启用哪些功能，为什么？*

以下两篇文章深入探讨了有关 Linux 内核功能的理论和实践最佳实践：

- [Linux 功能：它们为什么存在以及它们是如何工作的](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work) ：`https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work`
- [实践中的 Linux 功能](https://blog.container-solutions.com/linux-capabilities-in-practice) `https://blog.container-solutions.com/linux-capabilities-in-practice`

**防止Linux权限提升**

您应该在关闭权限提升的情况下运行容器，以防止使用`setuid`或`setgid`二进制文件提升权限。

#### 网络策略

1. **容器可以与网络中的任何其他容器通信**，并且在这个过程中没有地址转换——即不涉及 NAT。
2. **集群中的节点可以与网络中的任何其他容器通信，反之亦然**。即使在这种情况下，也没有地址转换——即没有 NAT。
3. **一个容器的 IP 地址总是相同的**，如果从另一个容器或它本身来看，它是独立的。

集群中的恶意用户要获得对集群的访问权限——他们可以向整个集群发出请求。

为了解决这个问题，您可以使用网络策略定义如何允许 Pod 在当前命名空间和跨命名空间中进行通信。

**启用网络策略**

Kubernetes 网络策略指定 Pod 组的访问权限，就像云中的安全组用于控制对 VM 实例的访问一样。

换句话说，它在 Kubernetes 集群上运行的 Pod 之间创建了防火墙。

 [Kubernetes 集群网络](https://ahmet.im/blog/kubernetes-network-policy/) : `https://ahmet.im/blog/kubernetes-network-policy/`

**每个名称空间中都应该有一个默认较保守的NetworkPolicy**

如果想深入了解[如何丢弃/限制运行在 Kubernetes 上的应用程序的流量](https://github.com/ahmetb/kubernetes-network-policy-recipes)，请查看此文档。 `https://github.com/ahmetb/kubernetes-network-policy-recipes`

#### 基于角色的访问控制（RBAC）策略

**禁用默认ServiceAccount的自动挂载**

请注意，[默认的 ServiceAccount 会自动挂载到所有 Pod 的文件系统中](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)。

您可能希望禁用它并提供更精细的策略。

**RBAC策略设置为所需的最少权限**

很难找到关于如何设置 RBAC 规则的好建议。在[Kubernetes RBAC 的 3 种实际方法中](https://thenewstack.io/three-realistic-approaches-to-kubernetes-rbac/)，您可以找到三个实际场景和有关如何开始的实用建议。

实际场景参考文档：`https://thenewstack.io/three-realistic-approaches-to-kubernetes-rbac/`

**RBAC策略是细粒度且不共享**

有一个简明的策略来定义角色和 ServiceAccounts。

首先，他们描述了他们的要求：

- 用户应该能够部署，但他们不应该被允许阅读 Secrets 例如
- 管理员应该获得对所有资源的完全访问权限
- 默认情况下，应用程序不应获得对 Kubernetes API 的写入权限
- 应该可以写入 Kubernetes API 以用于某些用途。

这四个要求转化为五个单独的角色：

- 只读
- 超级用户
- 操作员
- 控制器
- 行政

[您可以在此链接中](https://kubernetes-on-aws.readthedocs.io/en/latest/dev-guide/arch/access-control/adr-004-roles-and-service-accounts.html)了解他们的定义：`https://kubernetes-on-aws.readthedocs.io/en/latest/dev-guide/arch/access-control/adr-004-roles-and-service-accounts.html`

#### 自定义策略

**仅允许从已知的镜像仓库部署容器**

**在Ingress强制定义域名的唯一性**

**在 Ingress 主机名中使用已批准的域名**



### 集群配置

#### 推荐的Kubernetes配置

**集群通过CIS基准从测试**

互联网安全中心为保护您的代码的最佳实践提供了一些指南和基准测试。

他们还维护了kubernetes的基准；[官网地址](https://www.cisecurity.org/benchmark/kubernetes)：`https://www.cisecurity.org/benchmark/kubernetes`

虽然您可以阅读冗长的指南并手动检查您的集群是否合规，但更简单的方法是下载并执行[`kube-bench`](https://github.com/aquasecurity/kube-bench). 

下载地址：`https://github.com/aquasecurity/kube-bench`

[`kube-bench`](https://github.com/aquasecurity/kube-bench)是一种工具，旨在自动化 CIS Kubernetes 基准测试并报告集群中的错误配置。

示例输出：

```
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
[WARN] 1.1.1 Ensure that the --anonymous-auth argument is set to false (Not Scored)
[PASS] 1.1.2 Ensure that the --basic-auth-file argument is not set (Scored)
[PASS] 1.1.3 Ensure that the --insecure-allow-any-token argument is not set (Not Scored)
[PASS] 1.1.4 Ensure that the --kubelet-https argument is set to true (Scored)
[PASS] 1.1.5 Ensure that the --insecure-bind-address argument is not set (Scored)
[PASS] 1.1.6 Ensure that the --insecure-port argument is set to 0 (Scored)
[PASS] 1.1.7 Ensure that the --secure-port argument is not set to 0 (Scored)
[FAIL] 1.1.8 Ensure that the --profiling argument is set to false (Scored)
```

当master由云厂商托管的话，可能无法使用`kube-bench`。

**限制对alpha或beta API功能的访问**

Alpha 和 beta Kubernetes 功能正在积极开发中，可能存在导致安全漏洞的限制或错误。

始终评估 Alpha 或 Beta 功能可能提供的价值，以应对您的安全状况可能面临的风险。

#### 验证

**使用 OpenID (OIDC) 令牌作为用户身份验证策略**

Kubernetes 支持各种身份验证方法，包括 OpenID Connect (OIDC)。

OpenID Connect 允许单点登录 (SSO)（例如您的 Google 身份）连接到 Kubernetes 集群和其他开发工具。

您无需单独记住或管理凭据。

您可以将多个集群连接到同一个 OpenID 提供程序。

[有关 Kubernetes 中的 OpenID 连接的更多信息。](https://thenewstack.io/kubernetes-single-sign-one-less-identity/) 可以访问：`https://thenewstack.io/kubernetes-single-sign-one-less-identity/`

#### 基于角色的访问控制 (RBAC)

**ServiceAccount 令牌仅适用于应用程序和控制器**

服务帐户令牌不应用于尝试与 Kubernetes 集群交互的最终用户，但它们是在 Kubernetes 上运行的应用程序和工作负载的首选身份验证策略。

#### 日志记录设置

**日志的保留和归档策略**

您应该保留 30-45 天的历史日志。

**从节点、控制平面、审计收集日志**

从哪些方面收集日志：

- 节点（kubelet、容器运行时）
- 控制平面（API 服务器、调度程序、控制器管理器）
- Kubernetes 审计（对 API 服务器的所有请求）

你应该收集什么：

- 应用名称。从元数据标签中检索。
- 应用实例。从元数据标签中检索。
- 应用程序版本。从元数据标签中检索。
- 集群 ID。从 Kubernetes 集群中检索。
- 容器名称。从 Kubernetes API 中检索。
- 运行此容器的集群节点。从 Kubernetes 集群中检索。
- 运行容器的 Pod 名称。从 Kubernetes 集群中检索。
- 命名空间。从 Kubernetes 集群中检索。

**首选每个节点上的守护进程来收集日志而不是边车**

应用程序应该记录到标准输出而不是文件。

[每个节点上的守护进程可以从容器运行时收集日志](https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure)（如果记录到文件，可能需要每个 pod 的 sidecar 容器）。

参考地址：`https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure`

**配置日志聚合工具**

使用日志聚合工具，例如 EFK stack（Elasticsearch、Fluentd、Kibana）、DataDog、Sumo Logic、Sysdig、GCP Stackdriver、Azure Monitor、AWS CloudWatch。

