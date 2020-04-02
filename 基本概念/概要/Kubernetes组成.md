# Kubernetes组成
Kubernetes是个集群。

k8s集群包含一群工作机器，称为[节点]()，运行容器化的应用程序。每个集群至少要有一个工作节点。

工作节点上面运行着[Pod]()，而Pod又是应用程序负载的组成部分。[control plane]()负责管理工作节点，以及集群中的Pod们。生产环境中的control plane一般运行在多台服务器上，集群规模一般也是多个节点，也就可以实现容错和高可用。

本文列出了一个完整的Kubernetes集群所需的各个功能组件。

下图给出了这些组件如何互联互通。

![components-of-kubernetes](img/components-of-kubernetes.png)

- [Control Plane](#Control-Plane)
- [Node](#Node)
- [扩展组件](#扩展组件)
- [接下来……](#接下来)

## Control Plane
Control Plane为集群制定全局性的决策（比如调度），发现并响应集群事件（比如某个部署的`副本数`不满足需要，那就要启动一个新的[pod]()）。

Control Plane的各个组成部分可以运行在集群中的任意机器上。但是为了简单一些，我们的安装脚本一般都会把Control Plane的所有组件跑在同一台机器上，并且这个机器不会运行普通的用户容器。可以参考[构建高可用集群]()，了解如何部署多主节点的集群。
### kube-apiserver
API服务负责暴露Kubernetes API。相当于control plane的前端组件。

Kubernetes API服务的主要实现就是[kube-apiserver]()。kube-apiserver支持水平扩展——部署更多的实例即可。可以运行多个kube-apiserver实例，然后在上层做一个负载均衡。
### etcd
它是一个一致的、高可用的KV存储，k8s可以将全部的集群信息存储在etcd上。
如果k8s使用了etcd作为存储，那你最好为etcd做一个数据备份方案。
关于etcd详见[官方文档](https://etcd.io/docs)。
### kube-scheduler
本宝宝负责监控那些新建的，但是还没有分配[节点]()的[Pod]()，然后为它们选择一个节点。

调度过程中需要考量的因素包括：单个、多个Pod的资源需求，软硬件策略限制，亲和性、互斥性声明，是否依赖本地数据，工作负载之间是否有干扰，以及Pod的存活时间。
### kube-controller-manager
本宝宝负责运行各种[控制器]()进程。
逻辑上来说，每个[控制器]()是一个单独的进程，但为了降低复杂度，就把它们都编译到了一个二进制文件中了，运行在一个进程内。
这些控制器包括：
- 节点控制器（Node Controller）：当节点挂掉，负责发现并作出响应。
- 副本控制器（Replication Controller）：负责为系统中的每个副本控制器对象维护正确数量的Pod。
- 端点控制器（Endpoints Controller）：发布端点对象（把Service和Pod组织起来）。
- 服务账户和令牌控制器（Service Account & Token Controller）：为新的命名空间创建默认的账户和API访问令牌。
### cloud-controller-manager
[cloud-controller-manager]()运行着和底层云服务商打交道的控制器。从k8s的1.6版本开始作为内测功能加入。

cloud-controller-manager只运行云服务商特定的控制循环。可以通过kube-controller-manager来禁用这些循环：启动kube-controller-manager时将`--cloud-provider`设置为`external`。

cloud-controller-manager使得云服务商的代码和k8s的代码各自独立发展。在之前的版本中，k8s的代码和云服务商的代码存在一些功能上的耦合。在之后的版本中，云服务商的代码由他们自己维护，运行k8s的时候可以连接到cloud-controller-manager。

以下这些控制器都跟底层云服务商有些关联：

- 节点控制器：当某个节点停止响应的时候，通过检查云服务商来判断这个节点是否在云上已经被删除了。
- 路由控制器（Route Controller）：在底层云基础设施上设置路由。
- 服务控制器（Service Controller）：创建、更新、删除云服务商提供的负载均衡器。
- 数据卷控制器（Volume Controller）：创建、分配、挂载数据卷，和云服务商交互完成数据卷的编排工作。

## Node
Node组件运行在每个节点上，维护着运行中的pod，为k8s提供运行环境。

### kubelet

集群中每个[节点]()上的一个代理。它可以确保[容器]()是运行在[Pod]()里的。

kubelet手里攥着一堆从各种渠道过来的Pod定义（PodSpecs），确保定义中描述的容器都正常的运行。对于那些不是由k8s创建的容器，kubelet不管那些闲事儿。
### kube-proxy

kube-proxy是每个[节点]()上的网络代理，负责一部分k8s[服务]()概念下的功能。

[kube-proxy]()维护着节点上的网络规则。有了这些网络规则，集群上的Pod之间得以建立网络连接，集群外部的网络也得以和Pod进行通信。

如果操作系统本身拥有数据包过滤层，kube-proxy便会加以利用，否则它就自己处理各种流量转发工作。
### 容器运行时
容器运行时是用来运行容器的软件。

k8s支持的容器运行时包括：[Docker](https://docs.docker.com/engine/)、[containerd](https://containerd.io/docs/)、[CRI-O](https://cri-o.io/#what-is-cri-o)，以及其他实现了[Kubernetes CRT(Container Runtime Interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)接口的软件。

## 扩展插件

