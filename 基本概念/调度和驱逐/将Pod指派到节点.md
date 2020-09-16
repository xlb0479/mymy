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

给节点打标签可以让Pod定位到某些或者某一组节点上。这样可以保证特定的Pod只会运行在带有特定隔离、安全、或管控属性的节点上。当你带着这种目的使用标签时，强烈建议你使用那些不会被kubelet进程修改的标签key。

## 亲和性与反亲和性