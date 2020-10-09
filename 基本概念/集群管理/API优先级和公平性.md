# API优先级和公平性

**功能状态**：`Kubernetes v1.18 [alpha]`

在高负载场景下控制apiserver的行为，这是集群管理员的主要工作之一。[kube-apiserver](../概要/Kubernetes组成.md#kube-apiserver)有一些方式可以限制正在处理的工作量（比如`--max-requests-inflight`和`--max-mutating-requests-inflight`命令行参数），避免一大波请求直接把apiserver打垮，但是这些参数还不足以保证在高负载场景下让那些最重要的任务能够得到保障。

API优先级和公平性（API Priority and Fairness，APF）可以用来优化上面提到的工作量限制的问题。APF可以将请求以更细的粒度进行分类和隔离。它还引入了一些队列，保证在短时雷雨大风及强对流天气下请求不会被拒绝。请求在队列中按照公平算法进行分发，这样，比如一个捣乱的[Controller](../集群架构/控制器.md)就不会祸祸到其他的资源（甚至是统一优先级的）。

>**小心**：如果是“长时间运行”的请求——主要是watch请求——是不受APF过滤器控制的。即便没有开启APF，而是设置了`--max-requests-inflight`也是如此。

## 开启APF

APF特性门默认是关闭的。关于特性门的解释及如何开启和关闭，请看[这里](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)。APF特性门的名字叫“APIPriorityAndFairness”。该特性还涉及到一个[API组](../概要/Kubernetes%20API.md#API分组)也一定要开启。可以把下面的命令行参数加到`kube-apiserver`来实现这一切：

```shell script
kube-apiserver \
--feature-gates=APIPriorityAndFairness=true \
--runtime-config=flowcontrol.apiserver.k8s.io/v1alpha1=true \
 # …其他参数不用动
```

如果设置了`--enable-priority-and-fairness=false`，那就会关闭APF特性，即便其他的参数会开启它。

## 概念

在APF中又会涉及到几种不同的功能。进来的请求会使用*FlowSchema*按照请求的属性进行分类，赋予不同的优先级。优先级会产生不同的隔离度，维护单独的并发限制，这样不同优先级的请求就不会互相干扰。在同一优先级下，由公平队列算法保证不同的*flow*中的请求不会互相干扰，让请求进行排队，这样在短时雷雨大风及强对流天气中，请求依然不会失败，而且还能降低平均负载。

### 优先级

如果没有开启APF，是通过`kube-apiserver`的`--max-requests-inflight`和`--max-mutating-requests-inflight`来控制整体并发量的。当APF开启后，这些参数定义的并发量会进行求和，然后将求和后的值分配到指定数量的*优先级（priority level）* 上。每一个进来的请求都会赋予一个单独的优先级，每个优先级受控于它所配置的并发量。

在默认配置中，举个栗子来说，包含了leader选举请求、内置控制器请求以及来自Pod请求对应的优先级。这就是说，一个搞事情的Pod发出一大堆请求到apiserver，并不会干扰到leader选举或内置控制器的请求。

### 排队

即便是在同一个优先级下，也可能存在大量来源不同的请求。在超负荷场景中，限制某些请求避免其影响其他请求是非常重要的（特别是经常出现的那种有bug的客户端一股脑发出一大堆请求到apiserver，理想情况下这个客户端不应该对其他客户端产生明显的影响）。这块就是通过公平队列算法来处理同一个优先级下的请求的。每个请求赋予一个*flow*，由匹配的FlowSchema加上一个*flow distinguisher*来做标识——其结果要么是请求的用户，要么是目标资源的命名空间，要么什么也不是——然后系统会给同一优先级下不同flow中的请求赋予近似相等的权重。

当请求被分配到一个flow中之后，APF将请求分配一个队列中。这种分配策略使用了一种叫做[shuffle sharding](https://v1-18.docs.kubernetes.io/docs/reference/glossary/?all=true#term-shuffle-sharding)的技术，可以非常有效的从高强度flow中隔离出低强度的flow。

排序算法可以按照每个优先级进行调优，允许管理员在内存使用、公平性、短时流量激增以及队列所带来的延迟之间做出权衡。

### 特殊请求

有些请求特别重要，它会绕过该功能的限制。这种特别对待的方式可以保证一旦你的配置不对，不会导致apiserver彻底被关掉。

## 默认值

APF默认做好了一些推荐的配置，满足试验性的尝试；如果你的集群打算试试高负载场景，那你要考虑以下什么样的配置才是最好的。默认的配置将请求分成了五大优先级类型：

- `system`优先级，用于`system:nodes`组下的请求，即Kubelet，它是必须要跟apiserver交互的，这样才能调度工作负载。
- `leader-election`优先级，用于内置controller的leader选举请求（特别是，在`kube-system`命名空间中的`system:kube-controller-manager`或`system:kube-scheduler`用户和ServiceAccount发起的对`endpoints`、`configmaps`、`leases`的请求）。这些请求需要从其他流量中隔离出来，因为如果leader选举失败，会导致它们的控制器异常并重启，进而导致新的controller同步信息时产生更大更复杂的流量。
- `workload-high`优先级，用于内置controller的其他请求。
- `workload-low`优先级，用于其他ServiceAccount发起的请求，一般会包括Pod中运行的controller的全部请求。
- `global-default`优先级，处理所有其他流量，比如非特权用户使用`kubectl`进行交互。

此外，还有两个内置的PriorityLevelConfiguration和两个内置的FlowSchema，它们是无法被覆盖的：

- 特殊的`exempt`优先级，用于那些完全不受控的请求：它们总是会被立即分发。特殊的`exempt`FlowSchema会将所有`system:masters`组中的请求都分类到该优先级中。如果合适的话，你可以定义其他的FlowSchema将其他请求也分类到这个优先级中。
- 特殊的`catch-all`优先级，和特殊的`catch-all`FlowSchema配合使用，保证每个请求最终都能有一个分类。一般来说你不要指望它能为你干点儿什么，而是应该创建适合你自己的catch-all的FlowSchema和PriorityLevelConfiguration（或者使用内置的`global-default`配置）。为了能够捕获错误的配置导致某些请求没有被分类，该优先级只允许共享一套并发而且还不会给请求做队列，这就会导致如果流量只匹配了`catch-all`的FlowSchema就很有可能导致HTTP 429错误。

## 健康检查的特殊对待

在推荐的配置中，对于从本地kubelet向kube-apiserver发起的健康检查请求并不会有特殊对待——它们用了安全端口但是没有提供凭据信息。在这种配置下，这些请求被赋予了`global-default`FlowSchema以及对应的`global-default`优先级，会导致其他的请求把健康检查给挤出去。

如果你添加以下这种FlowSchema，可以让这些请求免于并发限制。

>**小心**：如果做了这种修改，那些不怀好意的人就可以发送匹配该FlowSchema的健康检查，想法多大发多大。如果你有web流量过滤或者类似的外部安全机制保护apiserver，可以添加一些特殊规则块，拒绝来自集群外部的健康检查请求。

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1alpha1
kind: FlowSchema
metadata:
  name: health-for-strangers
spec:
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: exempt
  rules:
  - nonResourceRules:
    - nonResourceURLs:
      - "/healthz"
      - "/livez"
      - "/readyz"
      verbs:
      - "*"
    subjects:
    - kind: Group
      group:
        name: system:unauthenticated
```

## 资源

流控API涉及了两种资源。[PriorityLevelConfiguration](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#prioritylevelconfiguration-v1alpha1-flowcontrol-apiserver-k8s-io)定义了隔离类型，可以处理的并发量比例，并且还可以对排队行为进行调优。[FlowSchema](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#flowschema-v1alpha1-flowcontrol-apiserver-k8s-io)用来对每个请求进行分类，每一个请求对应一个PriorityLevelConfiguration。

### PriorityLevelConfiguration

一个PriorityLevelConfiguration代表了一个隔离类型。每个这玩意都有一个单独的当前请求限制数，以及排队请求的限制数。

这玩意的并发限制并不以请求的绝对数值来定义，而是以“并发配额（concurrency shares）”。apiserver的总的并发限制数会按照这个配额比例分配给每一个PriorityLevelConfiguration。这样集群管理员就可以修改`--max-requests-inflight`（或`--max-mutating-requests-inflight`）并重启`kube-apiserver`来对整齐并发流量进行增大或减小，所有的PriorityLevelConfiguration都会基于他们自己的比例享受到上限值的增大（或减小）。

>**小心**：如果开启了API优先级和公平性，服务的总的并发限制等于`--max-requests-inflight`加上`--max-mutating-requests-inflight`。是否短时流量不再做区分；如果对于某个资源你想做区别对待，可以为对应的动词定义不同的FlowSchema。

当请求值大于对应PriorityLevelConfiguration设定的并发级别，它的`type`字段表示会对后续的请求做出怎样的回应。如果是`Reject`，那后续过量的请求就会立即被拒绝并返回HTTP 429（Too Many Requests）。如果是`Queue`那就是将请求放入队列，然后用混排以及公平队列算法在不同请求流中进行平衡。

在队列配置中，可以对公平队列算法进行调优。算法细节可以去看[增强提案](#下一步)，我们这里简单一说：

- 增大`queues`会减少不同flow之间的碰撞，但是会增加内存消耗。如果设成1，逻辑上相当于关闭了公平算法，但仍然允许请求被排队。
- 增大`queueLengthLimit`可以留住短时激增的请求，但是会增加响应延迟和内存消耗。
- 修改`handSize`可以调整不同flow间碰撞的几率，以及在过载情况下一个flow整体的并发情况。

>**注意**：`handSize`越大，两个flow碰撞的几率越小（因此会导致一个把另一个饿死），但同时可以减少apiserver上flow的数量。`handSize`越大同时还会增大一个大流量flow的响应延迟。一个flow排队的请求数上限为`handSize * queueLengthLimit`。

下表展示了一系列有趣的混排配置，并且对照了一只老鼠（低密度flow）被大象（高密度flow）踩死的几率，并列出不同大象数量下的几率。表格的算法见[https://play.golang.org/p/Gi0PLgVHiUg](https://play.golang.org/p/Gi0PLgVHiUg)。

**HandSize**|**Queues**|**1只大象**|**4只大象**|**16只大象**
-|-|-|-|-
12|32|4.428838398950118e-09|0.11431348830099144|0.9935089607656024
10|32|1.550093439632541e-08|0.0626479840223545|0.9753101519027554
10|64|6.601827268370426e-12|0.00045571320990370776|0.49999929150089345
9|64|3.6310049976037345e-11|0.00045501212304112273|0.4282314876454858
8|64|2.25929199850899e-10|0.0004886697053040446|0.35935114681123076
8|128|6.994461389026097e-13|3.4055790161620863e-06|0.02746173137155063
7|128|1.0579122850901972e-11|6.960839379258192e-06|0.02406157386340147
7|256|7.597695465552631e-14|6.728547142019406e-08|0.0006709661542533682
6|256|2.7134626662687968e-12|2.9516464018476436e-07|0.0008895654642000348
6|512|4.116062922897309e-14|4.982983350480894e-09|2.26025764343413e-05
6|1024|6.337324016514285e-16|8.09060164312957e-11|4.517408062903668e-07

### FlowSchema

一个FlowSchema匹配一堆请求并赋予它们一个优先级。每个进来的请求都会轮流被每一个FlowSchema进行测试，从数值最小的——也就是逻辑上的最大的——`matchingPrecedence`开始。找到第一个匹配的即可。

>**小心**：只有第一个匹配请求的FlowSchema才会生效。如果多个FlowSchema都能够匹配同一个请求，则取最高`matchingPrecedence`的那个。如果多个FlowSchema匹配同一个请求，并且`matchingPrecedence`也一样，则字典序`name`最小的获胜，但最好还是不要依赖这种情况，最好还是确保每个FlowSchema的`matchingPrecedence`都不一样。

当一个FlowSchema中的至少一条`rules`匹配了请求，那它就算是匹配成功了。一个rule匹配的前提是，它的`subjects`至少要匹配一个*并且*`resourceRules`或`nonResourceRules`（取决于请求的是资源还是非资源）至少要匹配一个。

对于subjects中的`name`字段，以及资源或非资源rule中的`verbs`、`apiGroups`、`resources`、`namespaces`、`nonResourceURLs`字段，通配符`*`都可以用来匹配对应字段下的所有值，相当于没有这个字段了。

FlowSchema的`distinguisherMethod.type`决定了匹配的请求会如何分到flow中。可以是`ByUser`，一个用户的请求就不会挤死另一个用户的请求，或者是`ByNamespace`，一个命名空间的请求就不会挤死另一个命名空间的请求，或者留空（或者直接删掉`distinguisherMethod`这个字段），这样所有匹配的请求都会分到同一个flow中。具体的选择要依赖于资源以及你的具体环境。

## 排错

开启了API优先级和公平性之后，apiserver给HTTP响应的时候，会带上两个额外的header：`X-Kubernetes-PF-FlowSchema-UID`和`X-Kubernetes-PF-PriorityLevel-UID`，分别标记了请求所匹配的FlowSchema和优先级。API对象的名字并不会包含在里面，用户有可能没有权限看到它们，所以当你进行排错工作的时候，可以用下面的命令：

```shell script
kubectl get flowschemas -o custom-columns="uid:{metadata.uid},name:{metadata.name}"
kubectl get prioritylevelconfigurations -o custom-columns="uid:{metadata.uid},name:{metadata.name}"
```

这样就可以得到FlowSchema和PriorityLevelConfiguration的UID和名字信息。

## 观测性

### 指标

当你开启了API优先级和公平性之后，kube-apiserver会暴露一些额外的指标。监控这些指标可以让你发现有问题的配置，避免重要的流量被卡脖子，还可以发现某些搞事情的工作负载正在祸祸你的系统。

- `apiserver_flowcontrol_rejected_requests_total`是一个Counter向量（从服务启动后开始不断累加），统计了被拒绝的请求数，维度按`flowSchema`标签（请求匹配的FlowSchema）、`priorityLevel`标签（请求被赋予的优先级）划分，还有`reason`标签。`reason`标签的值包含：
    - `queue-full`，表示队列中已经有太多的请求了，
    - `concurrency-limit`，表示配置了PriorityLevelConfiguration，而不是将请求进行排队，或者
    - `time-out`，表示当请求排队持续时间达到上限时依然处于队列中。
- `apiserver_flowcontrol_dispatched_requests_total`是一个Counter向量（从服务启动后开始不断累加），统计了开始处理的请求数，维度按`flowSchema`标签（表示请求匹配的FlowSchema）和`priorityLevel`标签（表示请求被赋予的优先级）划分。
- `apiserver_current_inqueue_requests`，它是一个Gauge向量，表示最近排队请求数的高水位线，按`request_kind`标签分组，其值可能是`mutating`或`readOnly`。这项高水位线标记代表了最近一秒钟时间窗口内完成的数量。这项指标跟以前的`apiserver_current_inflight_requests`Gauge向量形成了互补之势，后者代表了最近一个时间窗口内正在被响应的请求数的高水位线。
- `apiserver_flowcontrol_read_vs_write_request_count_samples`是一个Histogram向量，表示了当时请求数的观测值，维度包含`phase`标签（其值包括`waiting`和`executing`）和`request_kind`标签（其值包括`mutating`和`readOnly`）。这项观测值以较高频率进行周期性统计。
- `apiserver_flowcontrol_read_vs_write_request_count_watermarks`是一个Histogram向量，表示请求数的高或低水位线，维度包含`phase`标签（其值包括`waiting`和`executing`）和`request_kind`标签（其值包括`mutating`和`readOnly`）；`mark`标签的值只有`high`和`low`。此项水位线值的累计根据`apiserver_flowcontrol_read_vs_write_request_count_samples`的窗口而定，每当后者增加一个统计值，前者就统计一次。这些水位线值展示了每次采样之间的取值范围。
- `apiserver_flowcontrol_current_inqueue_requests`是一个Gauge向量，统计了排队中（尚未开始执行）请求数的瞬时值，维度包含`priorityLevel`和`flowSchema`标签。
- `apiserver_flowcontrol_current_executing_requests`是一个Gauge向量，包含了执行中（不是在队列中等待）请求数的瞬时值，维度包含`priorityLevel`和`flowSchema`标签。
- `apiserver_flowcontrol_priority_level_request_count_samples`是一个Histogram向量，包含了当时请求数的观测值，维度包含`phase`标签（其值包括`waiting`和`executing`）和`priorityLevel`标签。每项观测值都是周期性统计，从相关类型的上一次活动开始统计。观测值也是按较高频率统计的。
- `apiserver_flowcontrol_priority_level_request_count_watermarks`是一个Histogram向量，表示请求数的高或低水位线，维度包含`phase`标签（其值包括`waiting`和`executing`）和`priorityLevel`标签；`mark`标签的值只有`high`和`low`。此项水位线值的累计根据`apiserver_flowcontrol_priority_level_request_count_samples`的窗口而定，每当后者增加一个统计值，前者就统计一次。这些水位线值展示了每次采样之间的取值范围。
- `apiserver_flowcontrol_request_queue_length_after_enqueue`是一个Histogram向量，表示队列的长度，维度包括`priorityLevel`和`flowSchema`标签，从入队的请求中采样。每一个入队的请求都会为它的Histogram贡献一个采样，请求入队后就会报告一次队列的长度。注意相对于一个公平的测量，这项指标会给出不同的统计值。

>**注意**：这个Histogram中的异常值意味着某一个flow（也就是根据具体配置可能是一个用户或一个命名空间的请求）可能正在大量狂灌apiserver，并且已经出现瓶颈了。与之相比，如果某个优先级的Histogram显示它所有队列的长度都要比其他优先级的长，可能需要增加对应PriorityLevelConfiguration中的并发配额（concurrency share）了。

- `apiserver_flowcontrol_request_concurrency_limit`是一个Gauge向量，表示计算出来的并发限制（基于apiserver的总体并发限制和PriorityLevelConfiguration的并发配额），维度包括`priorityLevel`标签。
- `apiserver_flowcontrol_request_wait_duration_seconds`是一个Histogram向量，表示请求排队排了多久，维度包含`flowSchema`标签（表示请求匹配的FlowSchema）和`priorityLevel`标签（表示请求被赋予的优先级），还有`execute`标签（表示请求是否开始执行）。

>**注意**：因为每个FlowSchema会给请求赋予唯一一个PriorityLevelConfiguration，所以可以将某一个优先级的所有FlowSchema的Histogram值加起来，就等效于某个优先级的Histogram值了。

- `apiserver_flowcontrol_request_execution_seconds`是一个Histogram向量，表示请求的执行花了多长时间，维度包括`flowSchema`标签（表示请求匹配的FlowSchema）和`priorityLevel`标签（表示请求被赋予的优先级）。


## 调试接口

当你开启了API优先级和公平性之后，kube-apiserver会在它的HTTP(S)端口上额外增加以下路径。
- `/debug/api_priority_and_fairness/dump_priority_levels`——当前所有优先级及其状态的列表。可以这样获得：

```shell script
kubectl get --raw /debug/api_priority_and_fairness/dump_priority_levels
```

输出如下：

```text
PriorityLevelName, ActiveQueues, IsIdle, IsQuiescing, WaitingRequests, ExecutingRequests,
workload-low,      0,            true,   false,       0,               0,
global-default,    0,            true,   false,       0,               0,
exempt,            <none>,       <none>, <none>,      <none>,          <none>,
catch-all,         0,            true,   false,       0,               0,
system,            0,            true,   false,       0,               0,
leader-election,   0,            true,   false,       0,               0,
workload-high,     0,            true,   false,       0,               0,
```

- `/debug/api_priority_and_fairness/dump_queues`——所有队列及其当前状态的列表。可以这样获得：

```shell script
kubectl get --raw /debug/api_priority_and_fairness/dump_queues
```

输出如下：

```text
PriorityLevelName, Index,  PendingRequests, ExecutingRequests, VirtualStart,
workload-high,     0,      0,               0,                 0.0000,
workload-high,     1,      0,               0,                 0.0000,
workload-high,     2,      0,               0,                 0.0000,
...
leader-election,   14,     0,               0,                 0.0000,
leader-election,   15,     0,               0,                 0.0000,
```

- `/debug/api_priority_and_fairness/dump_requests`——当前所有在队列中排队等待的请求列表。可以这样获得：

```shell script
kubectl get --raw /debug/api_priority_and_fairness/dump_requests
```

输出如下：

```text
PriorityLevelName, FlowSchemaName, QueueIndex, RequestIndexInQueue, FlowDistingsher,       ArriveTime,
exempt,            <none>,         <none>,     <none>,              <none>,                <none>,
system,            system-nodes,   12,         0,                   system:node:127.0.0.1, 2020-07-23T15:26:57.179170694Z,
```

除了排队的请求，对于绕过（exempt）限制的优先级还包含了一行虚记录。

可以这样获取更详细的信息：

```shell script
kubectl get --raw '/debug/api_priority_and_fairness/dump_requests?includeRequestDetails=1'
```

输出如下：

```text
PriorityLevelName, FlowSchemaName, QueueIndex, RequestIndexInQueue, FlowDistingsher,       ArriveTime,                     UserName,              Verb,   APIPath,                                                     Namespace, Name,   APIVersion, Resource, SubResource,
system,            system-nodes,   12,         0,                   system:node:127.0.0.1, 2020-07-23T15:31:03.583823404Z, system:node:127.0.0.1, create, /api/v1/namespaces/scaletest/configmaps,
system,            system-nodes,   12,         1,                   system:node:127.0.0.1, 2020-07-23T15:31:03.594555947Z, system:node:127.0.0.1, create, /api/v1/namespaces/scaletest/configmaps,
```

## 下一步……

关于API优先级和公平性的更多背景知识，可以去看[增强提案](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190228-priority-and-fairness.md)。你可以通过[SIG API Machinery](https://github.com/kubernetes/community/tree/master/sig-api-machinery)提出建议或增加新的特性。