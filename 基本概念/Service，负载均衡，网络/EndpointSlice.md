# EndpointSlices

**功能状态**：`Kubernetes v1.17 [beta]`

*EndpointSlices*提供了一种简单的方式，可以跟踪集群内的网络端点（endpoint）。相对于Endpoint，它们的伸缩性和扩展性更强。

## 动机

Endpoint接口提供了一种粗暴的方式，用于跟踪k8s中的网络端点。很不幸（不幸个鸟），集群中的Service越来越多了，这个接口的限制也越来越明显。最显著的问题是，当要扩展到更大规模的网络端点时，挑战性也越来越大。

因为一个Service的所有网络端点都保存在了一个Endpoint资源中，那么这个资源可能变得非常之大。这就会影响到k8s的性能（尤其是主节点的control plane），当Endpoint发生变化时，会产生很大的网络流量，需要进行更多的处理工作。EndpointSlice可以减小这种问题带来的影响，并且还会其他特性提供了一个可扩展的平台，比如基于拓扑的路由。

## EndpointSlice资源

在k8s中，一个EndpointSlice包含了一组网络端点的引用。当定义了Service的[选择器](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)后，EndpointSlice控制器会自动为其创建EndpointSlice。这些EndpointSlice会包含所有跟Service选择器匹配的Pod。EndpointSlice根据唯一的Service和Port，将网络端点组织起来。EndpointSlice的对象名必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

下面是一个EndpointSlice栗子，它对应的Service名为`example`。

```yaml
apiVersion: discovery.k8s.io/v1beta1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    topology:
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```

默认情况下，由EndpointSlice控制器所管理的EndpointSlice，每个最多可以包含100端点（endpoint）信息。在这种规模范围内，EndpointSlice和Endpoint、Service时1比1的映射，并且具有相似的性能（这块没太翻译明白）。

当kube-proxy进行内网流量路由时，可以将EndpointSlice作为真实来源。一旦启用，在大量endpoint的场景下会带来不错的性能提升。

### 地址类型

EndpointSlice支持三种地址类型：

- IPv4
- IPv6
- FQDN（完全限定名）

### 拓扑

EndpointSlice中的每个endpoint都可以包含相关的拓扑信息。这个功能是用来指示endpoint到底应该在哪里的，包含了对应的节点、区域、地域信息。当这些值可用时，EndpointSlice控制器就会打上以下拓扑标签：

- `kubernetes.io/hostname` - endpoint所在的节点名
- `topology.kubernetes.io/zone` - endpoint所在的区域名
- `topology.kubernetes.io/region` - endpoint所在的地域名

这些标签的值都是来自和endpoint相关的资源。hostname标签就来自对应Pod的NodeName字段。zone和region标签则是来自Node的同名标签。

### 管理

默认情况下由EndpointSlice控制器来创建和管理EndpointSlice。有很多其他不同的情况，比如服务编排（service mesh）的场景下，一部分EndpointSlice就可能是由其他实体或控制器来管理的。为了保证它们之间不会互相干扰，EndpointSlice中的`endpointslice.kubernetes.io/managed-by`标签就是用来指明这个EndpointSlice是由谁控制的。如果是EndpointSlice控制器，那么它会给这个标签赋值为`endpointslice-controller.k8s.io`。其他的实体在进行管理的时候也是要打上唯一的标签值。

### 归属

绝大多数情况下EndpointSlice都应该归属于对应的Service。这个关系可以在EndpointSlice的`kubernetes.io/service-name`标签中查到。

## EndpointSlice控制器

EndpointSlice控制器需要监视Service和Pod，保证EndpointSlice的状态是最新的。控制器为每个Service管理它们的EndpointSlice。为匹配的Pod实现IP端点。

### EndpointSlice的容量

默认情况下每个EndpointSlice最多可以包含100个endpoint。可以在[kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)上面配置`--max-endpoints-per-slice`来修改这个值，最大1000。

### EndpointSlice的分布

每个EndpointSlice都包含了一堆端口，用于其中的endpoint。当Service中使用了命名port，那么同一个命名port对应的Pod的目标端口可能是不同的，这也就需要不同的EndpointSlice。其中的逻辑跟endpoint的分组是类似的（隔得时间有点久，没太明白咋回事）。

控制器会尽量把EndpointSlice都装满，但是不会经常进行重新分配。控制器的想法十分简单：

- 1.遍历当前所有的EndpointSlice，删除已经不需要的endpoint，更新那些发生变更的endpoint。
- 2.遍历上一步中所有被修改的EndpointSlice，将需要新建的endpoint都装进去。
- 3.如果还有剩下的endpoint，试着把它们放到那些未曾改变的EndpointSlice中，或者创建新的EndpointSlice。

其中的关键点是，第三步可以在一个分布很完美的EndpointSlice中提前限制对EndpointSlice进行更新。举个例子，比如要增加10个新的endpoint，现在一共有两个EndpointSlice，每个都有大于5个的剩余空间，按照我们的逻辑，此时会创建一个新的EndpointSlice而不是填到当前的两个中。换句话说，如果要更新多个EndpointSlice，那么会优先考虑创建一个新的EndpointSlice。

因为每个节点的kube-proxy会监视着EndpointSlice，每一次EndpointSlice更新的代价就会高一些，因为它会影响到集群的所有节点。我们的这种方法有意识地限制了需要对每个节点进行通知的次数，即便这可能会导致出现很多没有填满的EndpointSlice。

实际情况中这种不理想的分布应该是比较少见的。控制器处理的大部分变更都比较小，可以装到已经存在的EndpointSlice中，如果不行，那就很可能需要一个新的EndpointSlice了。Deployment的滚动更新会自然地将EndpointSlice重新打包，所有的Pod和它们的endpoint都可以顺利替换。

## 下一步……

- [开启EndpointSlice](https://kubernetes.io/docs/tasks/administer-cluster/enabling-endpointslices/)
- 学习[通过Service来连接应用](通过Service来连接应用.md)