# Pod生命周期

本文为您介绍Pod的生命周期，看我，多有礼貌，为您！啧啧~

- [各个阶段](#各个阶段)
- [Condition](#Condition)
- [容器探针](#容器探针)
- [Pod和容器的Status](#Pod和容器的Status)
- [容器状态](#容器状态)
- [可读性](#可读性)
- [重启策略](#重启策略)
- [我能活多久](#我能活多久)
- [栗子](#栗子)
- [接下来……](#接下来)

## 各个阶段

Pod的`status`属性是一个[PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podstatus-v1-core)对象，包含一个`phase`字段。

Pod所处的每一个阶段（phase），是对它整个生命周期的一个简单的、宏观的描述。每个阶段并不是对容器或Pod状态的汇总，不能当作综合状态机。（有点学术了？）

Pod所处的阶段数量有严格的限制。除了本文提到的，绝对不要假设Pod的`phase`还有其他值。如果有人跟你说还有XXX，你就当他是个骗子。

下面是`phase`的所有可能值：

值|描述
-|-
`Pending`|k8s系统已经收到这个Pod了，但是容器的镜像还没创建好。这段时间包含了调度所需的时间，以及从网络拉取镜像的时间，可能得耗一阵子。
`Running`|Pod跟节点的关系已经建立起来了，容器也都创建好了。至少有一个容器处于运行或启动中、重启中的状态。
`Succeeded`|Pod中所有容器都成功地完成了任务并结束了，并且不会被重启。
`Failed`|Pod中所有容器都结束了，至少有一个容器的结束状态有问题。也就是说，容器的退出码不等于0，或者是被系统操作系统杀掉了。
`Unknown`|出于某种原因，无法获取到Pod的状态，一般是由于无法跟Pod所在的主机建立通信导致的。

## Condition

每个Pod有一个PodStatus，里面是一个[PodCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podcondition-v1-core)的数组，Pod可能通过，也可能没有通过这些Condition。PodCondition数组中的每个元素包含六个可能的字段：

- `lastProbeTime`，该condition最后一次探测的时间戳。
- `lastTransitionTime`，记录Pod最后一次status改变的时间戳。
- `message`，人类可读的该condition的详细信息。
- `reason`，一个唯一的驼峰命名单词，记录上一次condition发生变化的原因。
- `status`，一个字符串，可能的值包括“`True`”、“`False`”、“`Unknown`”。
- `type`，一个字符串，可能值包括：
    - `PodScheduled`：Pod已经被调度到一个节点上了；
    - `Ready`：Pod已经可以开始接流量了，所有匹配的Service都可以将其加入到负载均衡池中了；
    - `Initialized`：所有[初始化容器]()已经正常启动；
    - `ContainersReady`：Pod中所有容器都准备好了。