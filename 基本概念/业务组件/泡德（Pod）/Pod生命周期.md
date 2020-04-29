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
`Pending`|asd