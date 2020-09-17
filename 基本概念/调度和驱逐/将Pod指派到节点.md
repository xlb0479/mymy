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