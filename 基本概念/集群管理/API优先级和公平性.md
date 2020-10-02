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