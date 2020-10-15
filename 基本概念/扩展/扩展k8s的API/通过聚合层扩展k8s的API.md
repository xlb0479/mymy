# 通过聚合层扩展k8s的API

聚合层可以为k8s扩展出核心API之外的新的API。这些新增的API可以作为[Service目录](../Service目录.md)现成的解决方案，或者是你自己开发一些API。

聚合层不同于[自定义资源](自定义资源.md)，后者是让[kube-apiserver](../../概要/Kubernetes组成.md#kube-apiserver)认识新的对象类型。

## 聚合层

聚合层运行在kube-apiserver进程内。除非有扩展资源注册进来，否则聚合层啥也不干。要想注册一个API，你要添加一个*APIService*对象，它要“声明”它在k8s API中的URL路径。然后，聚合层就会将针对该API路径（比如`/apis/myextension.mycompany.io/v1/…`）的所有操作都代理到注册的APIService上。

实现APIService的最常用的方式就是在Pod中跑一个*扩展API服务（extension API server）*。如果你使用了扩展API服务来管理集群资源，那么这个扩展API服务（还可以写作“扩展apiserver”）通常会跟一个或多个[控制器](../../集群架构/控制器.md)一起配合工作。apiserver-builder库中提供了扩展API服务以及相关控制器的基本骨架。

### 响应延迟

扩展API服务跟kube-apiserver之间的网络应当做到低延迟。kube-apiserver的网络发现请求应当在五秒甚至更短时间内就能打一个来回。

如果你的扩展API服务无法达到这种延迟要求，那你得想想办法，看看改改什么东西。你可以设置kube-apiserver的`EnableAggregatedDiscoveryTimeout=false`[特性门](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)来关闭超时限制。后续的版本中会删除这个已经废弃的特性门。

## 下一步……

- 让聚合器在你的环境中跑起来吧，[配置一下聚合层](https://v1-18.docs.kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)。
- 然后为聚合层[部署一个扩展api-server](https://v1-18.docs.kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/)。
- 还有，学学如何[使用CRD扩展k8s API](https://v1-18.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)。
- 看一下[APIService](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#apiservice-v1-apiregistration-k8s-io)的规范。