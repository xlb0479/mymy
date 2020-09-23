# k8s调度器

在k8s中，*调度*就是确保[Pod](../业务组件/泡德（Pod）/Pod.md)能跟[节点](../集群架构/节点（Node）.md)关联上，然后[kubelet](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)就可以运行它们了。

## 概要

调度器会监视那些新创建还没有关联节点的Pod。调度器要负责为它发现的每一个Pod找到一个合适的下家。调度决策需要考虑到下面列出的调度原则。

如果你想知道为什么Pod被放到了一个特定节点上，或者你准备实现一个自定义的调度器，本文会帮你了解到调度相关的知识。

## kube-scheduler

[kube-scheduler](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)是k8s的默认调度器，作为[control plane](https://v1-18.docs.kubernetes.io/docs/reference/glossary/?all=true#term-control-plane)的一部分来运行。kube-scheduler经过精心设计，如果你想的话，可以编写你自己的调度组件来替代它。

对于每个新建的Pod，或者其他还没有被调度的Pod，kube-scheduler会为其选择一个最佳节点来运行。但是Pod中的每个容器对于资源有不同的需求，而且每个Pod也有不同的需求。因此当前的所有节点在调度的时候需要根据具体需求来进行过滤。

在一个集群中，满足Pod需求的节点被称为 *可用的（feasible）* 节点。如果没有合适的节点，Pod就一直处于未调度的状态直到调度器能给它找到下家。

调度器为一个Pod找到所有合适的节点，然后运行一组函数来给这些节点打分，从中选择分值最高的节点来运行Pod。然后调度器通知apiserver，这个过程叫做*绑定（binding）*。

调度决策中要考量的因素包括单独的以及整体的资源需求，硬件/软件/策略约束，亲和性、反亲和性声明，数据本地化，组件内部冲突等等。

### kube-scheduler的节点选择

kube-scheduler为Pod选择一个节点拢共分两步：

- 1.过滤
- 2.打分
- 3.把冰箱门儿带上

*过滤*这一步是为Pod找到合适的节点。比如PodFitsResources过滤器会检查备选节点的资源是否满足Pod中定义的资源请求值（request）。这一步完成后，节点列表中包含了所有合适的节点；通常要大于一个。如果得到的节点列表为空，那Pod目前就还无法进行调度。

在*打分*这一步中，调度器要对选出的节点进行排名，选择Pod最理想的下家。根据当前的打分规则，调度器会给每一个选出的节点打一个分值。

最后，kube-scheduler将Pod许配给分值最高的那个节点。如果相同分值的节点数量大于一个，那就随机选一个。

有两种方式可以对调度器的过滤和打分行为进行配置：

- [调度策略](https://v1-18.docs.kubernetes.io/docs/reference/scheduling/policies/)允许你配置对过滤的*期望（Predicates）*，以及对打分的*优先级（Priorities）*。
- [调度配置](https://v1-18.docs.kubernetes.io/docs/reference/scheduling/profiles/)允许你配置插件来实现不同的调度阶段，包括：`QueueSort`、`Filter`、`Score`、`Bind`、`Reserve`、`Permit`等等。还可以让kube-scheduler运行不同的配置。

## 下一步……

- 阅读[调度器的性能调优](调度器的性能调优.md)
- 阅读[Pod拓扑约束](../业务组件/泡德（Pod）/Pod拓扑约束.md)
- 阅读kube-scheduler的[参考文档](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
- 学习[配置多个调度器](https://v1-18.docs.kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
- 学习[拓扑管理策略](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
- 学习[Pod损耗](../配置/Pod损耗.md)