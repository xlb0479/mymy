# LimitRange

默认情况下容器在k8s中可用的[计算资源](../配置/管理容器资源.md)是不受限的。有了资源配额，管理员就可以在每个[命名空间](../概要/Kubernetes对象/命名空间.md)的基础上去限制资源的创建和使用。在一个命名空间中，Pod或容器可用的CPU和内存要受限于命名空间的资源配额。不能让某个Pod或者容器把所有资源都吃光了。LimitRange就是一种在命名空间中限制资源分配（给Pod或容器）的策略。

*LimitRange*可以：

- 约束一个命名空间中每个Pod或容器可用的计算资源的最大和最小值。
- 约束一个命名空间中每个PVC可请求（request）的容量的最大和最小值。
- 约束一个命名空间中一个资源的请求（request）上限（limit）比。
- 为一个命名空间设置默认的请求/上限参数，运行时自动注入到容器中。

## 开启LimitRange

从1.10版本开始就默认启用了LimitRange。

只有在一个命名空间中创建一个LimitRange对象后，它的约束才会生效。

LimitRange对象的名字必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

### 概要

- 管理员选择命名空间并创建一个LimitRange。
- 用户在这个命名空间下创建Pod、容器、PVC。
- 对于那些没有设置计算资源需求的Pod和容器，`LimitRanger`准入控制器（admission controller）强制执行默认策略，限制并跟踪观察Pod和容器的资源使用情况，保证它们不会超过所在命名空间下LimitRange定义的最大和最小值，以及请求上限比。
- 如果创建或更新了一个资源（Pod、容器、PVC）导致超出了LimitRange的约束，请求apiserver时会返回HTTP`403 FORBIDDEN`，同时还会告诉你违反了怎样的约束。
- 如果LimitRange对某个计算资源比如`cpu`和`memory`做出了限制，那用户就必须要设置对应资源的请求或上限值。否则系统会拒绝创建Pod。
- LimitRange校验只会发生在Pod准入（Admission）阶段，不会影响已经在运行中的Pod。

几个LimitRange的栗子：

- 2节点集群，总共8G内存16核，限制一个命名空间下的Pod请求值为100m的CPU，上限为500m，内存请求为200Mi，上限600Mi。
- 对于没有设置cpu和内存请求参数的容器，定义默认的CPU请求和上限值为150m，内存的默认请求值为300Mi。

如果一个命名空间的限制总量少于其中Pod/容器的上限之和，就会有资源争抢。那就不能再创建容器或者Pod了。

LimitRange的各种变化都不会影响到已经创建好的资源。

## 下一步……

看看[LimitRanger设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/admission_control_limit_range.md)，了解更多信息。

几个栗子：

- [如何为每个命名空间设置最大和最小CPU约束](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)。
- [如何为每个命名空间设置最大和最小内存约束](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)。
- [如何为每个命名空间设置默认的CPU请求和上限值](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)。
- [如何为每个命名空间设置默认的内存请求和上限值](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)。
- [如何为每个命名空间设置存储使用的最大和最小值](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/#limitrange-to-limit-requests-for-storage)。
- 一个[配额设置的详细栗子](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)。