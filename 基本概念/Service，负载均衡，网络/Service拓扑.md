# Service拓扑

**功能状态**：`Kubernetes v1.17 [alpha]`

*Service拓扑*可以让Service根据集群节点的拓扑来转发流量。比如可以将Service的流量优先转发到跟客户端在同一个节点或可用区内的Endpoint上。

## 简介

默认情况下，打到`ClusterIP`或`NodePort`类型的Service上的流量会被路由到任意的后端地址上。在1.7版本中已经可以将“外部”的流量路由到节点的Pod上（原意可能不是这样，实在想不明白），但是不支持`ClusterIP`类型的Service，同样，更复杂的拓扑——比如区域化路由——也是不支持的。而*Service拓扑*的出现解决了这些问题，可以基于节点标签定义