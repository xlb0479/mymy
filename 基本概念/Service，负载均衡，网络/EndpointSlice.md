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

默认情况下，由EndpointSlice控制器所管理的EndpointSlice，每个最多可以包含100端点（endpoint）信息。在这种规模范围内，EndpointSlice和Endpoint、Service时1比1的映射，并且具有相似的性能（这块每太翻译明白）。

