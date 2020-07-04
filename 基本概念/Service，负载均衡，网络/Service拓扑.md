# Service拓扑

**功能状态**：`Kubernetes v1.17 [alpha]`

*Service拓扑*可以让Service根据集群节点的拓扑来转发流量。比如可以将Service的流量优先转发到跟客户端在同一个节点或可用区内的Endpoint上。

## 简介

默认情况下，打到`ClusterIP`或`NodePort`类型的Service上的流量会被路由到任意的后端地址上。在1.7版本中已经可以将“外部”的流量路由到节点的Pod上（原意可能不是这样，实在想不明白），但是不支持`ClusterIP`类型的Service，同样，更复杂的拓扑——比如区域化路由——也是不支持的。而*Service拓扑*的出现解决了这些问题，可以基于流量两端节点标签定义流量路由策略。

通过流量两端节点的标签匹配，操作员可以定义出哪些节点“较远”、哪些“较近”，完全由操作员自己去考量。对于公有云上的操作员们来说，大部分流量都希望保持在同一个可用区里，因为跨可用区的传输总是有更多代价的。其他常见的场景包括将流量路由到DaemonSet管理的本地Pod上，或者是为了得到更低的延迟，将流量保持在同一个顶层交换机下。

## 使用Service拓扑

如果集群开启了Service拓扑，你可以通过Service中的`topologyKeys`字段来实现流量路由控制。这个字段是一个节点标签的有序列表，访问Service时，会基于这个列表对Service下的Endpoint进行排序。根据该字段的顺序，流量会被路由到和来源节点持有同样标签的节点上，如果没有匹配就找下一个标签，直到所有标签都检查了一遍。

如果没有找到匹配的节点，流量就会被拒绝，效果就跟Service没有后端Pod是一样的。也就是说，Endpoint的选择是先要找到第一个跟`topologyKey`匹配的节点，并且该节点还要有可用的Pod。如果定义了这个字段但是没有找到匹配的后端，对于当前的客户端来说，这个Service就不存在可用的后端，那么连接就应该失败。特殊值`"*"`用来表示“任意的拓扑”。它会匹配所有的值，如果要用，都是放在整个列表的最后一位上。

如果`topologyKey`没有定义，或者是空的，那就不会有拓扑限制。

假设集群中的节点上都打好了主机名、可用区、地域等标签。然后你可以如下设置`topologyKeys`来定义流量路由策略。

- 只发给同一个节点的Endpoint，没有的话就失败：`["kubernetes.io/hostname"]`。
- 优先发给同一个节点，不行就发给同一个可用区的，再不行就同一个地域的，再不行就完犊子吧：`["kubernetes.io/hostname", "topology.kubernetes.io/zone", "topology.kubernetes.io/region"]`。这种方式在数据本地化要求严格的场景下非常有用。
- 优先同一个可用区的，否则发给其他任意的Endpoint：`["topology.kubernetes.io/zone", "*"]`。

## 约束

- Service拓扑和`externalTrafficPolicy=Local`无法兼容，所以Service无法同时使用这两个功能。
- 有效的`topologyKeys`目前只有`kubernetes.io/hostname`、`topology.kubernetes.io/zone`和`topology.kubernetes.io/region`，未来还会支持更多节点标签。
- `topologyKeys`必须是有效的标签key，最多可以定义16个。
- 如果要用通配的`"*"`，必须放在最后一位。

## 栗子

下面是几个Service拓扑的用例。

### 仅限节点本地的Endpoint

这是一个只会路由到节点本地Endpoint的Service。如果节点上没有Endpoint，流量会被拒绝：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
```

### 优先节点本的Endpoint

这个例子是优先节点本的Endpoint，否则发到其他任意的Endpoint上：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "*"
```

### 仅限统一可用区或地域

优先发到同一个可用区，其次是同一个地域。如果都没有，流量被拒绝。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
```

### 优先节点本地，其次是可用区，再次是地域

优先发送到节点本地，其次可用区，再次同一地域，否则就任意选一个Endpoint。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```

## 下一步……

- 看看[如何开启Service拓扑]()
- 看看[通过Service来连接应用](通过Service来连接应用.md)