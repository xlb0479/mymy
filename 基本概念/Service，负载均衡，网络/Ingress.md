# Ingress

**功能状态**：`Kubernetes v1.1 [beta]`

用来管理外部对集群内Service访问的API对象，一般是HTTP。

Ingress可以提供负载均衡、SSL终结以及基于名字的虚拟主机。

## 术语

为了超级清晰，我们定义以下术语：

- 节点（Node）：k8s中的一个机器，属于集群的一部分。
- 集群（Cluster）：由k8s管理，一组运行容器化应用的节点。最常见的k8s部署方式是，所有集群节点都是在内网的。
- 边界路由（Edge router）：为集群执行防火墙策略的路由器。它可能是一个由云服务商管理的网关，或者一个物理硬件设备。
- 集群网络（Cluster network）：一组链接，逻辑的或者是物理的，基于k8s的[网络模型](../集群管理/集群网络.md)，为集群建立通信功能。
- 服务（Service）：一个k8s的[Service](Service.md)，用[标签](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)选择器标识了一组Pod。除非特殊提及，我们都假设Service使用的是虚拟IP，只能在集群网络中进行路由。