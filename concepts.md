# 基础知识
本章就是让你了解一下Kubernetes的各个零部件，整个[集群](https://www.baidu.com/)是由啥构成的，深入理解Kubernetes是如何运作的。
- [概要](#概要)
- Kubernetes 对象
- Kubernetes Control Plane
- 接下来……
## 概要
当你用Kubernetes的时候，你需要使用*Kubernetes API对象*来描述你要*让集群达到一个什么样的状态*：运行什么东西，用什么镜像，跑几个副本，需要什么样的网络环境和磁盘资源，等等。通过创建各种Kubernetes API 对象，你就可以设定你所需要的目标状态，一般都是通过命令行工具`kubectl`来完成这些设定。当然你也可以直接调用Kubernetes API来完成这些工作。
当你闹完上面的操作之后，*Kubernetes Control Plane*会通过Pod Lifecycle Event Generator([PLEG](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md))，让整个集群达到你所设定的目标状态。在这个过程中，Kubernetes会偷偷的完成大量的工作，比如启动/重启容器，扩展应用的副本数，等等。Kubernetes Control Plane由以下几个进程组成：

- **Kubernetes Master**，它由三个进程组成，并且都运行在同一个机器上，这个机器自然也就被称为主节点。三个进程分别是：[kube-apiserver]()，[kube-controller-manager]()以及[kube-scheduler]()。
- 每个非主节点上面跑两个进程：
   - [kubelet]()，跟Kubernetes Master进行通信。
   - [kube-proxy]()，网络代理，为每个节点提供网络服务。
## Kubernetes objects
Kubernetes系统包含这么几个抽象的概念：容器化的应用及负载，相关的网络及磁盘资源，以及集群的其他信息。这些抽象的概念都被Kubernetes API实现成了一个一个的对象。参见[理解Kubernetes对象]()。基本的Kubernetes对象包括：
- [泡德Pod]()
- [Service]()
- [Volume]()
- [Namespace]()
基于这些基础的对象，Kubernetes通过[控制器]()又封装出了一些高级的功能，包括：
- [Deployment]()
- [DaemonSet]()
- [StatefulSet]()
- [ReplicaSet]()
- [Job]()
