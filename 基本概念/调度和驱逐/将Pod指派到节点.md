# 将Pod指派到节点

可以将[Pod](../业务组件/泡德（Pod）/Pod.md)约束在某些[节点](../集群架构/节点（Node）.md)上，或者优先运行在某些节点上。具体可以有多种方式来实现，推荐使用[标签选择器](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)来实现。一般来说不需要这种约束，因为调度器会自动选择一个合理的位置（比如将Pod均匀分配，避免分配个资源不足的节点等等），但是也有些情况你可能需要更详细地控制Pod的位置，比如要保证Pod能落到一个带有SSD的机器上，或者是让两个不同Service但是交互非常频繁的Pod能够落到同一个可用区里面。

## nodeSelector

`nodeSelector`是实现节点选择约束的最简单的方式。`nodeSelector`是PodSpec中的一个字段。定义了一个kv对儿的映射。要想让Pod能够运行在这个节点上，节点必须要有对应的kv标签（当然也可以有别的标签）。最常用的是一个kv对儿。

让我们一起来践踏这个栗子，看看怎么用`nodeSelector`。

### 第零步：前提条件

本例假设你已经了解了Pod的基本概念，并且已经[建立了一个k8s集群](https://v1-18.docs.kubernetes.io/docs/setup/)。

### 第一步：给节点加标签

执行`kubectl get nodes`获取集群节点的名字。选择要打标签的节点并执行`kubectl label nodes <node-name> <label-key>=<label-value>`。比如我的节点名是“kubernetes-foo-node-1.c.a-robinson.internal”，我要打的标签是“disktype=ssd”，那么最终的命令就是`kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd`。

可以执行`kubectl get nodes --show-labels`来确认节点标签是否正确。或者执行`kubectl describe node "nodename"`查看节点的完整标签列表。

### 第二步：给Pod添加nodeSelector

随便拿一个Pod配置文件，添加nodeSelector。比如我的Pod配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

然后添加一个nodeSelector：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

当你执行`kubectl apply -f https://k8s.io/examples/pods/pod-nginx.yaml`，Pod就会被调度到你打了标签的那个节点上。可以执行`kubectl get pods -o wide`查看Pod所在的“NODE”。

## 插曲：内置的节点标签

除了你[添加](#第一步：给节点加标签)的节点标签，节点已经预先设置了一套标准的标签。它们是

- [kubernetes.io/hostname](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-hostname)
- [failure-domain.beta.kubernetes.io/zone](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesiozone)
- [failure-domain.beta.kubernetes.io/region](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesioregion)
- [topology.kubernetes.io/zone](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)
- [topology.kubernetes.io/region](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)
- [beta.kubernetes.io/instance-type](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#beta-kubernetes-io-instance-type)
- [node.kubernetes.io/instance-type](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#nodekubernetesioinstance-type)
- [kubernetes.io/os](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os)
- [kubernetes.io/arch](https://v1-18.docs.kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-arch)

>**注意**：这些标签的值都依赖具体的云服务商，并不保证可靠。比如`kubernetes.io/hostname`的值在某些情况下可能跟主机名一样，但另一个场景中可能又变了。

## 节点隔离/限制

给节点打标签可以让Pod定位到某些或者某一组节点上。这样可以保证特定的Pod只会运行在带有特定隔离、安全、或管控属性的节点上。当你带着这种目的使用标签时，强烈建议你使用那些不会被kubelet进程修改的标签。这样可以避免节点通过kubelet的凭据给节点对象打上这些标签，进而影响调度器把工作负载（wordload）调度到这个节点上。（最后这段话没明白）

`NodeRestriction`准入控制器可以阻止kubelet去设置或修改带有`node-restriction.kubernetes.io/`前缀的标签。如果要使用这种前缀的标签来做节点隔离：

- 确保你使用了[节点授权](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/node/)并*开启*了[NodeRestriction准入控制器](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)。
- 使用`node-restriction.kubernetes.io/`前缀给节点打标签，然后在节点选择器中使用这些标签。比如`example.com.node-restriction.kubernetes.io/fips=true`或者`example.com.node-restriction.kubernetes.io/pci-dss=true`。

## 亲和性与反亲和性

`nodeSelector`可以通过非常简单的方式将Pod约束在带有特定标签的节点上。而亲和性与反亲和性进一步扩展了这些约束。关键点在于

- 1.亲和性与反亲和性的语法更加具有表现力，可入围奥斯卡奖的那种。这种语法不光提供精确匹配，在其基础之上还能提供逻辑AND操作。
- 2.你可以将规则定义为“软蛋”或“优先考虑”规则，而不是强制规则，这样一旦调度器无法得到满足，Pod依然可以被调度。
- 3.可以针对节点上运行的其他Pod的标签来进行约束（或者其他的拓扑域），而不仅仅针对节点的标签，这样就可以让Pod和Pod之间产生相斥或相吸。

亲和性分两部分，“节点亲和性”和“Pod间亲和性/反亲和性”。节点亲和性类似于`nodeSelector`（但是具有上面列出的前两条优势），Pod间亲和性/反亲和性则是针对Pod的标签进行约束，如上面第三条说的那样，同时还具有前两条优势。

### 节点亲和性

节点亲和性概念上类似于`nodeSelector`——允许你通过节点标签来约束Pod可以被调度到哪些节点上。

目前有两种节点亲和性，分别是`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`。你可以把它俩简单的认为是“硬核”跟“软蛋”，因为前者对于一个Pod要想调度到一个节点上*必须*满足其声明的规则（跟`nodeSelector`一样，但是语法更具有表现力，透视观比较好，人物刻画的也不错），而后者则意味着*优先考虑*，调度器会尽力满足，但并不保证一定。名字中“IgnoredDuringExecution”的含义同样也是跟`nodeSelector`的原理类似，如果一个节点在运行时修改了标签导致Pod的亲和性规则不再满足，那么这个Pod依然会运行在这个节点上。未来我们还打算实现`requiredDuringSchedulingRequiredDuringExecution`，跟`requiredDuringSchedulingIgnoredDuringExecution`类似，但是它会将不满足亲和性约束的Pod从节点上剔除掉。

因此比如一个`requiredDuringSchedulingIgnoredDuringExecution`的栗子，可能就是说“这个Pod只能运行在带有Intel CPU的节点上”，再比如一个`preferredDuringSchedulingIgnoredDuringExecution`的栗子，可能是在说“让这些Pod运行在XYZ可用区中，但如果实在不行的话，也可以运行在别的地方”。

节点亲和性是定义在PodSpec中的`affinity`里面的`nodeAffinity`字段中。

下面是一个使用节点亲和性的栗子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

上面这份节点亲和性规则指明，Pod只能被调度到节点标签key为`kubernetes.io/e2e-az-name`且值为`e2e-az1`或`e2e-az2`的节点上。此外，在满足该规则的基础上，优先考虑那些带有标签key为`another-node-label-key`且值为`another-node-label-value`的节点。

你看到我们在栗子中使用的`In`操作符。新的节点亲和性语法支持这些操作符：`In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`。可以通过`NotIn`和`DoesNotExist`来实现节点反亲和性，或者使用[冷屁股](热脸和冷屁股.md)让Pod远离某些节点。

如果你同时定义了`nodeSelector`和`nodeAffinity`，Pod要想调度到一个节点上，就必须两个条件*都*满足才行。

过你的`nodeAffinity`中定义了多个`nodeSelectorTerms`，那么当**其中一个**`nodeSelectorTerms`满足时，Pod就可以被调度到这个节点上。

如果你的`nodeSelectorTerms`中定义了多个`matchExpressions`，那**只有当全部**`matchExpressions`都满足的情况下Pod才能调度到这个节点上。

如果你修改或者删除了节点上的标签，已经存在的Pod是不会被删除的。换句话说，亲和性的判定是发生在Pod调度的时候。

`preferredDuringSchedulingIgnoredDuringExecution`中的`weight`字段的取值范围是1-100。对于能够满足所有调度要求（资源请求、RequiredDuringScheduling亲和性表达式等等）的节点，调度器会遍历该字段的所有元素，如果节点匹配对应的MatchExpressions那就再加上“weight”。然后把这个分值跟其他优先级函数的结果结合起来。最终得分最高者获得本次比赛的冠军。

### Pod间亲和性和反亲和性

Pod间亲和性与反亲和性可以让你*基于那些已经运行在节点上的Pod的标签*来约束Pod的位置，而不是直接基于节点本身的标签。规则大概就是说“这个Pod应当（如果是反亲和性那就是不应当）运行在X上，并且X上已经运行了一个或者多个满足规则Y的Pod”。规则Y通过节点选择器来表达，而且还可以加上命名空间；和节点不同的是，Pod是基于命名空间的（因此Pod上面的标签也是基于命名空间的），对于Pod进行标签选择的时候必须要指明选择器要应用于哪个命名空间。因此X就可以是一个拓扑域，比如节点、机架、可用区、地域等等。写的时候要使用`topologyKey`，也就是系统用来标明节点拓扑域的标签key；比如在[插曲：内置的节点标签](#插曲：内置的节点标签)中列出的key。

>**注意**：Pod间亲和性与反亲和性需要执行大量的处理逻辑，大大规模集群中会明显降低调度的效率。不建议在好几百个节点的集群中使用它们。

>**注意**：Pod反亲和性需要节点的标签保持一致，换句话说就是集群中的每个节点都应该有能够匹配`topologyKey`的标签。如果一些或者全部节点都没有`topologyKey`指定的标签，那将会导致不可预料的事情发生，比如蓝屏、花瓶、电视突然没信号了。

跟节点亲和性类似，Pod的亲和性与反亲和性也有两种，叫做`requiredDuringSchedulingIgnoredDuringExecution`以及`preferredDuringSchedulingIgnoredDuringExecution`，同样分别代表“软蛋”和“硬核”要求。可以看一下前面关于节点亲和性的介绍。比如`requiredDuringSchedulingIgnoredDuringExecution`亲和性就是说“将ServiceA和ServiceB的Pod放在一起，因为它们的交互比较多”，又比如`preferredDuringSchedulingIgnoredDuringExecution`反亲和性就是说“将这个Service的Pod均匀分布在各个可用区上”（这里如果用硬核要求的话就不太讲理了，因为你的Pod数量很有可能大于可用区数量）。

Pod间亲和性定义在PodSpec中的`affinity`里的`podAffinity`字段。而Pod间反亲和性则是定义在PodSpec中的`affinity`里的`podAntiAffinity`字段。

#### 使用Pod亲和性的栗子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: failure-domain.beta.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```

这里定义了一个亲和性规则和一个反亲和性规则。本例中，`podAffinity`是`requiredDuringSchedulingIgnoredDuringExecution`，`podAntiAffinity`是`preferredDuringSchedulingIgnoredDuringExecution`。其中亲和性规则就是说Pod所在的节点必须是在同样的可用区中并且已经至少有一个标签key为“security”值为“S1”的Pod运行在上面了。（更准确地讲，Pod可以运行在节点N，前提是节点N要有一个key为`failure-domain.beta.kubernetes.io/zone`的标签，其值为V，这样集群中就至少要有一个标签key为`failure-domain.beta.kubernetes.io/zone`且值为V的节点，然后这个节点上要运行着一个标签key为“security”且值为“S1”的Pod。）其中的反亲和性规则是说Pod不能被调度到那些已经运行了标签key为“security”且值为“S2”Pod的节点所在的可用区上。可以去看看[设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md)，里面有很多Pod亲和性与反亲和性的栗子，`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`的都有。

合法的操作符有`In`、`NotIn`、`Exists`、`DoesNotExist`。

原则上来说，`topologyKey`可以是任意合法的标签key。但是从性能和安全角度考虑，这里有一些限制：

- 1.对于亲和性，`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`都不允许空的`topologyKey`。
- 2.对于反亲和性，同样`requiredDuringSchedulingIgnoredDuringExecution`和`preferredDuringSchedulingIgnoredDuringExecution`都不允许空的`topologyKey`。
- 3.对于`requiredDuringSchedulingIgnoredDuringExecution`的反亲和性，准入控制器`LimitPodHardAntiAffinityTopology`会限制`topologyKey`必须是`kubernetes.io/hostname`。如果你想用自定义的拓扑，那就要修改准入控制器，或者关闭它。
- 4.除了上面列出的条件，其他合法的标签key都是可以的。

除了`labelSelector`和`topologyKey`，你还可以设置一个`namespaces`列表，用来声明`labelSelector`应用的命名空间（这个字段跟`labelSelector`和`topologyKey`在同一级）。如果省略该列表或者为空，那就跟Pod定义的命名空间一起走。

不论是亲和性还是反亲和性，`requiredDuringSchedulingIgnoredDuringExecution`的所有`matchExpressions`必须全部满足才能让Pod调度到该节点上。

#### 更实际的栗子

当用在比如ReplicaSet、StatefulSet、Deployment等其他高级集合中时，Pod间亲和性与反亲和性就变得更加有用了。可以方便的将一组工作负载定位到同一个拓扑中，比如同一个节点上。

##### 总是定位到同一个节点

在一个三节点集群中，web应用同样有内存缓存，比如Redis。我们希望让web服务器能够尽可能的跟缓存挨在一起。

下面这段yaml就定义了一个简单的Redis的Deployment，包含了三个副本，标签选择器为`app=store`。Deployment中定义了`PodAntiAffinity`，保证调度器不会让三个副本都调度到同一个节点上去。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

下面是web服务器的Deployment定义，包含了`podAntiAffinity`和`podAffinity`。这样就会告诉调度器，所有的副本必须要跟带有`app=store`标签的Pod在一起。同时还可以保证每个web服务器副本不会定位到同一个节点上。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

当我们创建了上面这两个Deployment，我们的三节点集群就会像下面这个样子。

**节点1**|**节点2**|**节点3**
-|-|-
*webserver-1*|*webserver-2*|*webserver-3*
*cache-1*|*cache-2*|*cache-3*

看看，`web-server`的3个副本自动的和缓存搭配在了一起。

```shell script
kubectl get pods -o wide
```

输出如下：

```text
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
redis-cache-1450370735-6dzlj   1/1       Running   0          8m        10.192.4.2   kube-node-3
redis-cache-1450370735-j2j96   1/1       Running   0          8m        10.192.2.2   kube-node-1
redis-cache-1450370735-z73mh   1/1       Running   0          8m        10.192.3.1   kube-node-2
web-server-1287567482-5d4dz    1/1       Running   0          7m        10.192.2.3   kube-node-1
web-server-1287567482-6f7v5    1/1       Running   0          7m        10.192.4.3   kube-node-3
web-server-1287567482-s330j    1/1       Running   0          7m        10.192.3.2   kube-node-2
```

##### 绝不定到同一个节点上

上面的栗子用了`PodAntiAffinity`规则，并且有`topologyKey: "kubernetes.io/hostname"`，用于部署Redis集群，这样就不会存在某两个实例定位到同一个节点上的情况了。见[ZooKeeper教程]()