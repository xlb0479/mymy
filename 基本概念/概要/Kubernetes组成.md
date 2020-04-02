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
