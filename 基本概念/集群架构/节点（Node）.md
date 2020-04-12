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
- [容量和剩余配额](#容量和剩余配额)
- [其他信息](#其他信息)

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

```json
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