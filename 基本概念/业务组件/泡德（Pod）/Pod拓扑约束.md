# Pod拓扑约束

**功能状态**：`Kubernetes v1.18`（beta）

可以通过*拓扑约束（topology spread constraints）*来控制Pod在集群中分布情况，涉及地域、区域、节点等其他用户定义的领域。这种工具可以提高可用性，提升整体的资源利用率。

- [前置条件](#前置条件)
- [拓扑约束](#拓扑约束)
- [和Pod亲和性的对比](#和Pod亲和性的对比)
- [已知的限制](#已知的限制)

## 前置条件

### 打开特性门

必须为[apiserver]()**和**[调度器]()打开`EvenPodsSpread`[特性门]()。

### 节点标签

拓扑约束基于节点标签来识别节点所在的拓扑领域。比如某个节点可能含有如下标签：   
`node=node1,zone=us-east-1a,region=us-east-1`

假设一个4节点的集群如下：

```text
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

集群的逻辑视图就成了下面这个吊样子：

```text
+---------------+---------------+
|     zoneA     |     zoneB     |
+-------+-------+-------+-------+
| node1 | node2 | node3 | node4 |
+-------+-------+-------+-------+
```

除了手动打标签，还可以重用一些[大众情人标签]()，这些标签在好多集群上已经都自动加好了。

## 拓扑约束

### API

`pod.spec.topologySpreadConstraints`字段是打1.16版本才有的：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
```

可以设置多个`topologySpreadConstraint`来引导kube-scheduler对Pod的调度。其中的字段包括：

- **maxSkew**用来描述Pod到底能有多么的不平均。它用来设置某个拓扑类型下，两个不同的拓扑域中Pod数量允许相差的最大值。必须大于0。
- **topologyKey**则是节点标签的Key。如果两个节点都有这个key，并且连值也一样，则调度器会把它俩认为是在相同的拓扑内。调度器会保持每个拓扑域中Pod数量的平衡。
- **whenUnsatisfiable**用来处理当Pod不满足拓扑约束时该怎么办：
    - `DoNotSchedule`（默认），让调度器停止调度这个Pod。
    - `ScheduleAnyway`，调度仍然继续，但是优先选择能让数量倾斜最小化的节点。
- **labelSelector**用来匹配适用的Pod。匹配到的Pod就会计入到对应拓扑域的Pod数量里。关于标签选择器可以看[这儿](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)。

有个查看字段详情的命令呦~   
`kubectl explain Pod.spec.topologySpreadConstraints`

### 例子：一个TopologySpreadConstraint

假设你有一个4节点集群，其中有3个带有`foo:bar`标签的Pod分别分配到了节点1、2、3上（`P`代表Pod）：

```text
+---------------+---------------+
|     zoneA     |     zoneB     |
+-------+-------+-------+-------+
| node1 | node2 | node3 | node4 |
+-------+-------+-------+-------+
|   P   |   P   |   P   |       |
+-------+-------+-------+-------+
```

如果现在要让后续的Pod能在两个Zone之间平均一些，那就应该像下面这样定义：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

`topologyKey: zone`就是说要在带有“zone:”标签的节点上平均分配。`whenUnstaisfiable: DoNotSchedule`就是告诉调度器，如果当前要分配的Pod无法满足拓扑约束，那就先别分配，等着。

如果调度器把当前的Pod调度到“zoneA”里，那分布情况就会变成[3,1]，导致倾斜值等于2（3-1），也就不满足`maxSkew: 1`。本例中，当前Pod只能分配到“zoneB”。

```text
+---------------+---------------+      +---------------+---------------+
|     zoneA     |     zoneB     |      |     zoneA     |     zoneB     |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
| node1 | node2 | node3 | node4 |  OR  | node1 | node2 | node3 | node4 |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
|   P   |   P   |   P   |   P   |      |   P   |   P   |  P P  |       |
+-------+-------+-------+-------+      +-------+-------+-------+-------+
```

可以调整Pod的spec来满足不同需要：

- 调大`maxSkew`，比如2，这样当前的Pod也能调度到zoneA上。
- 将`topologyKey`改为“node”，这样Pod就是在节点之间平均分配，而不是zone之间。如果是上面的例子，`maxSkew`仍然是“1”，那新的Pod就只能放到“node4”上。
- 将`whenUnsatisfiable: DoNotSchedule`改为`whenUnsatisfiable: ScheduleAnyway`，这样不会影响调度的进行（假设其他要求都满足）。但是它会优先把Pod放到匹配数量最少的拓扑域中。（注意这里仍然需要跟标准的调度策略共同决定，比如资源的利用率等。）

### 例子：多个TopologySpreadConstraint

本例还是基于上面的例子做一些修改。初始情况不变：

```text
+---------------+---------------+
|     zoneA     |     zoneB     |
+-------+-------+-------+-------+
| node1 | node2 | node3 | node4 |
+-------+-------+-------+-------+
|   P   |   P   |   P   |       |
+-------+-------+-------+-------+
```

但现在你要用两个拓扑约束，对区域和节点都要进行约束。

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

此时，要满足第一个约束，新的Pod就只能放到“zoneB”中；要满足第二个约束，新的Pod只能放到“node4”上。两个约束是“且”的关系，所以最后只能放在“node4”上。

多个约束可能会导致冲突。比如是3节点的集群，有2个zone：

```text
+---------------+-------+
|     zoneA     | zoneB |
+-------+-------+-------+
| node1 | node2 | node3 |
+-------+-------+-------+
|  P P  |   P   |  P P  |
+-------+-------+-------+
```

如果还用上面的约束，你会发现“mypod”会一直停留在`Pending`状态。这是因为，要满足第一个约束，“mypod”就得放在“zoneB”里；而要满足第二个约束，“mypod”就得放在“node2”上。它俩合起来的结果就是啥也得不到。

为了避免这种情况出现，要么调大`maxSkew`，要么把其中一个约束改为`whenUnsatisfiable: ScheduleAnyway`。

### 几个约定

注意几个隐式的约定：

- 只匹配同一个命名空间下的Pod。
- 节点如果没有`topologySpreadConstraints[*].topologyKey`指定的东西那就会被跳过。这就意味着：
    - 1.这些节点上的Pod不会影响`maxSkew`的计算——在上面的例子中，假设“node1”没有“zone”标签，那它的两个Pod就不会被计算，因此新的Pod就可以调度到“zoneA”里。
    - 2.新的Pod不会被调度到这种节点上——在上面的例子中，假设有个“node5”，标签为`{zone-typo: zoneC}`，这个节点就会被无视，因为没有“zone”标签。
- 如果说新的Pod中`topologySpreadConstraints[*].labelSelector`跟Pod自身的标签不匹配，那就可以被调度到“zoneB”里，因为此时约束条件是满足的。但是，其结果就是，集群的倾斜程度依然没有变——zoneA依然是有两个带{foo:bar}标签的Pod，zoneB中也依然是只有一个带{foo:bar}标签的Pod。如果这跟你想象的不一样，我们建议工作组件的`topologySpreadConstraints[*].labelSelector`能跟自身的标签匹配一下。
- 如果新的Pod有`spec.nodeSelector`或`spec.affinity.nodeAffinity`，不满足它们的节点会被忽略。

假设有5个节点，从zoneA到zoneC：

```text
+---------------+---------------+-------+
|     zoneA     |     zoneB     | zoneC |
+-------+-------+-------+-------+-------+
| node1 | node2 | node3 | node4 | node5 |
+-------+-------+-------+-------+-------+
|   P   |   P   |   P   |       |       |
+-------+-------+-------+-------+-------+
```

此时你知道“zoneC”是要排除在外的。按照下面的yaml，“mypod”会被分配到“zoneB”而不是“zoneC”。`spec.nodeSelector`的作用依然存在。

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: NotIn
            values:
            - zoneC
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

### 集群的默认约束

**功能状态**：`Kubernetes v1.18`（alpha）

可以设置集群的默认拓扑约束。默认的拓扑约束生效，当且仅当Pod：

- 没有在`.spec.topologySpreadConstraints`中定义任何约束。
- Pod属于某个服务（service）、副本控制器（replication controller）、副本集（replica set）或状态集（stateful set）。

默认约束可以作为[调度profile]()的`PodTopologySpread`插件参数的一部分。约束定义的方式跟[上面的API](#API)一样，除了`labelSelector`必须为空。选择器会根据Pod所属的服务、副本控制器、副本集或状态集来定。

示例如下：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha2
kind: KubeSchedulerConfiguration

profiles:
  pluginConfig:
    - name: PodTopologySpread
      args:
        defaultConstraints:
          - maxSkew: 1
            topologyKey: failure-domain.beta.kubernetes.io/zone
            whenUnsatisfiable: ScheduleAnyway
```

>**注意**：默认调度约束产生的评分可能会跟[`DefaultPodTopologySpread`插件]()的评分产生冲突。所以，如果想给`PodTopologySpread`设置默认约束，那就要在调度Profile中把这个插件关了。

## 和Pod亲和性的对比

在k8s中用“亲和性”来控制Pod的调度——更紧凑或者更分散。

- 对于`PodAffinity`，可以让Pod在匹配的拓扑域中更加的紧凑
- 对于`PodAntiAffinity`，每个拓扑域中只有会一个Pod。

“平均分配”的功能为Pod在普通拓扑域中的平均分配提供了灵活的选择——可以提高可用性，也可以降低成本。这一点在工作组件的滚动更新和副本平滑扩充时也是有帮助的。可以在[动机](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/20190221-pod-topology-spread.md#motivation)中窥探到其中的奥妙。

## 已知的限制

在1.18中，这个功能还是Beta阶段，有一些已知的限制：

- 减小Deployment的规模时，可能会导致Pod倾斜。
- 节点冷屁股（taint）的问题。见[Issue 80921](https://github.com/kubernetes/kubernetes/issues/80921)