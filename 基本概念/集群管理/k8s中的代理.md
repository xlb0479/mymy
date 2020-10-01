# k8s中的代理

本文介绍k8s中用到的代理。

## 代理

使用k8s的时候，你会接触到以下几种代理：

- 1.[kubectl代理](https://v1-18.docs.kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api)：
    - 运行在用户的桌面或者一个Pod中
    - 从localhost代理到apiserver
    - 客户端到代理之间使用HTTP
    - 代理到apiserver之间使用HTTPS
    - 定位apiserver
    - 添加认证header
- 2.[apiserver代理](https://v1-18.docs.kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#discovering-builtin-services)：
    - 是apiserver内置的一种堡垒机制
    - 将集群外部的用户连接到cluster IP上，否则可能根本无法访问
    - 运行在apiserver的进程中
    - 客户端到代理之间使用HTTPS（或者改一下apiserver用HTTP也行）
    - 代理到目标之间根据代理掌握的情报，用HTTP或HTTPS都有可能
    - 可以用来访问一个节点、Pod或Service
    - 用来访问Service的时候会做负载均衡
- 3.[kube proxy](../Service，负载均衡，网络/Service.md#serviceip)：
    - 运行在每个节点上
    - 代理UDP、TCP和SCTP
    - 它不懂HTTP
    - 提供负载均衡
    - 只用来访问Service
- 4.在apiserver之上的代理/负载均衡器：
    - 每个集群的存在和实现方式都不一样（比如nginx）
    - 位于所有的客户端和所有的apiserver之间
    - 如果有多个apiserver，它会充当负载均衡的作用。
- 5.位于外部服务的云负载均衡器：
    - 由一些云服务商提供（比如AWS ELB、Google Cloud Load Balancer）
    - 由`LoadBalancer`类型的Service自动创建
    - 通常只支持UDP/TCP
    - 是否支持SCTP要看具体的云服务商对负载均衡器的实现方式
    - 每个云服务商的实现都不一样

除了头两种以外，k8s用户一般都不怎么需要担心这些东西。集群管理员会确保后面几种类型的正确部署。

## 请求重定向

代理替代了重定向功能。重定向已经被废弃了。