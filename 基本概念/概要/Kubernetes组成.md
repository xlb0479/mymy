# Kubernetes组成
Kubernetes是个集群。
k8s集群包含一群工作机器，称为[节点]()，运行容器化的应用程序。每个集群至少要有一个工作节点。
工作节点上面运行着[Pod]()，而Pod又是应用程序负载的组成部分。[control plane]()负责管理工作节点，以及集群中的Pod们。生产环境中的control plane一般运行在多台服务器上，集群规模一般也是多个节点，也就可以实现容错和高可用。
本文列出了一个完整的Kubernetes集群所需的各个功能组件。
下图给出了这些组件如何互联互通。

![components-of-kubernetes](img/components-of-kubernetes.png)

- [Control Plane](#Control-Plane)
- [节点](#节点)
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
cloud-controller-manager使得云服务商的代码和k8s的代码各自独立发展。在之前的版本中，k8s的代码和云服务商的代码存在一些功能上的沟壑。在之后的版本中，云服务商的代码由他们自己维护，运行k8s的时候可以连接到cloud-controller-manager。