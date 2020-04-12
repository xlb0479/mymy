# 节点

一个节点就是k8s中的一个工作机（工作🐥？工作🐟？），行话叫`minion`。节点可以是虚拟机或物理机。每个节点包含运行[pod]()所需的各种服务，由master统一管理。这些服务包括[容器运行时](../概要/Kubernetes组成.md#容器运行时container-runtime)，kubelet和kube-proxy。详见[k8s节点](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node)。

- [节点状态](#节点状态)
- [节点管理](#节点管理)
- [节点拓扑](#节点拓扑)
- [API对象](#API对象)
- [接下来……](#接下来)

## 节点状态

节点的状态包含以下信息：

- [Addresses](#Addresses)
- [Conditions](#Conditions)
- [Capacity和Allocatable](#Capacity和Allocatable)
- [Info](#Info)

节点状态及其他信息可以用下面的命令来查看：

```text
kubectl describe node <这里填节点名>
```

输出的结果下面进行分别介绍。

### Addresses

这个字段的用法视你的云服务商或裸金属服务器的配置而定。

- HostName：主机名由节点内核给出。可以用kubelet的`--hostname-override`参数来重定义。
- ExternalIP：可以由外部路由的IP（从集群外访问）。
- InternalIP：集群内可路由的IP。

### Conditions

`conditions`字段描述所有`Running`的节点状态信息。包括：

节点Condition|描述
-|-
`Ready`|`True`代表节点处于健康状态，并且准备好运行pod了，`False`代表节点有问题，不能接受pod，`Unknown`代表节点控制器在过去`node-monitor-grace-period`（默认40秒）时间内没有收到节点的信息
`MemoryPressure`|`True`代表节点内存有压力了——也就是可用内存不多了；否则为`False`
`PIDPressure`|`True`代表进程压力——也就是当前进程太多了；否则为`False`
`DiskPressure`|`True`代表磁盘空间有压力了——也就是当前磁盘容量不多了；否则为`False`
`NetworkUnavailable`|`True`代表节点的网络配置有问题；否则为`False`

节点的condition是个JSON。下面是一个健康节点的响应信息。

```text
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```

如果`Ready`字段的值保持在`Unknown`或`False`超过`pod-eviction-timeout`（[kube-controller-manager]()的一个参数），这个节点的节点控制器就要开始删除节点上所有的Pod了。这个值默认是**五分钟**。某些情况下，节点失联了，apiserver无法与这个节点的kubelet通信。要删除pod这件事儿就无法传达给kubelet，直到重新建立连接。在这段时间内，这些将死的pod有可能还继续运行在这个失联的节点上。

在1.5版本之前，节点控制器会将这些失联的pod从apiserver中[强制删除]()。但是从1.5开始，节点控制器不会再去强删pod了，除非能够确认这些pod已经停止运行了。这些失联节点上的pod可能会处于`Terminating`或`Unknown`的状态。如果是k8s无法从底层判断出一个节点是否真的完全离开了集群，那么集群的管理员可以手动删除这个节点。删除节点会导致节点上的所有Pod对象从apiserver中清除，名字也释放了。（等我死了，我想把我的名字给你）

节点生命周期控制器会自动创建[冷屁股]()。调度器为节点指派Pod的时候，也会看看节点的冷屁股。Pod可以设置自己的热脸来贴到节点的冷屁股上。

### Capacity和Allocatable

描述节点资源的可用情况：CPU、内存以及可承受的pod数量。

Capacity一栏下的这些字段标识节点拥有的资源总数。Allocatable一栏下的这些字段表示节点可用的资源数量。

学习[保留计算资源]()一节的时候，你会对这部分知识有更深刻的理解。

### Info

节点的一般信息，比如内核版本，k8s版本（kubelet和kube-proxy版本），Docker版本（如果用了的话，没用就别琢磨了），以及操作系统的名字。这些信息都是Kubelet从节点上收集来的。

## 节点管理

跟[Pod]()和[Service]()不一样，节点不是k8s建的：节点是由外部的云服务商，比如Google Compute Engine来创建的，或者是你的什么物理机或虚拟机啊这种东西。因此我们说的k8s创建节点，其实是创建了一个用来代表节点的对象。建完之后，k8s要检查节点有没有生效。比如你用下面这段信息来创建一个节点：

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

k8s会在内部创建节点对象（节点的替身），然后根据`metadata.name`字段的值来进行健康检查。如果节点能用——所有的服务都已正常启动——那就可以在上面跑Pod了。否则任何集群活动都不会跟它产生关系直到它变得可用。节点对象的名字必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

>**注意**：对于无效的节点，k8s也会把对应的对象一直留着，并且持续进行检查，看它是否活过来了。要想停止这个过程，必须显式的把它删了。

目前有个三个组件会跟k8s的节点接口打交道：节点控制器、kubelet和kubectl。

### 节点控制器

节点控制器是master的一个组件，管理节点相关的各个方面。

在节点的一生中，节点控制器扮演了多个角色（戏精上身？）。首先，是在节点注册时，为其分配一个CIDR块（如果开启了CIDR分配）。

其次，要将节点控制器内部的节点列表，跟云服务商那边的可用机器列表保持一致。运行在云上的时候，如果某个节点闹妖了，节点控制器就会询问云服务商那边，看看这个节点的虚拟机是不是还能用。如果不能用了，节点就要把它从列表中删了。

第三，监视节点的健康状态。如果节点失联（比如节点挂了，节点控制器收不到心跳），节点控制器要负责将节点NodeStatus从NodeReady更新成ConditionUnknown，如果一直失联，就要开始从节点上删除所有的Pod（优雅关闭）。（判定为ConditionUnknown默认的超时时间时40秒，且在5分钟后开始删除Pod）节点控制器每`--node-monitor-period`秒检查一次节点状态。

#### 心跳

心跳，由节点发出，用于判定节点是否可用。有两种心跳：`NodeStatus`更新，和[租约对象](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#lease-v1-coordination-k8s-io)。每个节点在`kube-node-lease`命名空间下都有一个租约（Lease）对象。租约是一个轻量级的对象，旨在随着集群的扩展，优化心跳的性能。

Kubelet负责创建和更新`NodeStatus`以及租约对象。

- 当节点状态发生变化，或者在一个配置好的时间间隔内没有更新，kubelet就要对`NodeStatus`。默认的`NodeStatus`更新时间间隔为5分钟（比默认40秒的节点失联超时要长很多很多很多）。
- kubelet创建租约对象后每10秒（默认值）更新一次。租约的更新和`NodeStatus`的更新没有什么关系。如果租约更新失败，kubelet会根据指数退避算法进行重试，初始为200毫秒，每次加7秒。

#### 可靠性

在1.4版本中，我们更新了节点控制器在大量节点和主节点通信出现问题（主节点的网络出问题了）时的处理逻辑，以优化在此场景下的处理方法。从1.4开始，如果要剔除一个pod，节点控制器需要观察集群中所有的节点的状态。

大部分情况下，节点控制器会将剔除的速度控制在每秒`--node-eviction-rate`（默认0.1）个，也就是每10秒内，被剔除Pod的节点数不超过1个。

如果说节点闹妖是发生在某个可用区域中，那此时节点剔除的逻辑就不太一样了。节点控制器此时要看一下指定区域内闹妖（Ready状态为Unknown或False）的节点数量占比。如果占比超过`--unhealthy-zone-threshold`（默认0.55），那么剔除速率会进一步降低：如果是小集群（节点数量小于等于`--large-cluster-size-threshold`，默认50）则停止剔除，否则剔除速度降为`--secondary-node-eviction-rate`（默认0.01）个每秒。之所以对每个可用区执行这样的策略，是因为一个可用区从集群中割裂的时候，其他可用区依然保持正常，如果集群没有跨越多个云服务商的可用区，那整个集群就只有一个可用区。

将节点分布在多个可用区的主要原因之一，就是当某个可用区整段垮掉的时候，其上的工作负载可以转移到正常的可用区上。因此，当一个可用区内的所有节点都开始闹妖，那节点控制器就会以一般的`--node-eviction-rate`速度来进行剔除工作。极限情况是所有可用区都完犊子了（整个集群就没个正常的节点）。这种情况下，节点控制器会认为是主节点的网络出了什么问题，会停止剔除工作，直到能够恢复一些通信。

从1.6版本开始，节点控制器还要剔除那些运行在冷屁股（Taints）设置为`NoExecute`节点上的Pod，且这些Pod没有为节点的冷屁股设置热脸（Tolerates）。此外还有一个处于测试阶段且默认关闭的功能，节点控制器要根据节点的问题，比如失联或还没准备好，来为节点设置相应的冷屁股。详见[介个文档]()。

从1.8版本开始，可以让节点控制器来创建代表节点condition的冷屁股。这也是1.8的测试功能。