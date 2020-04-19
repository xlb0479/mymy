# 深入CloudControllerMananger

CloudControllerManager（CCM）概念（不要跟二进制文件搞混了），最初是为了让云服务商专有代码和k8s核心功能解耦的。CCM要跟其他的Master组件协作完成任务，比如KubernetesControllerManager、apiserver以及调度器。也可以作为k8s的插件来运行，运行在整个k8s的顶层。

CCM的设计理念就是插件化，通过插件，让新的云服务商快速同k8s整合。目前我们已经在承接新的云服务商，并且准备将一些旧的云服务商迁移到新的CCM模型上。

本文将介绍CCM的底层概念及相关功能。

下图是没有CCM的情况。

![pre-ccm-arch](img/pre-ccm-arch.png)

- [设计思路](#设计思路)
- [CCM组件](#CCM组件)
- [CCM的功能](#CCM的功能)
- [插件化](#插件化)
- [授权](#授权)
- [云服务商实现](#云服务商实现)
- [集群管理](#集群管理)

## 设计思路

在上图中，k8s和云服务商通过多个组件进行关联：

- Kubelet
- KubernetesControllerManager
- Apiserver

CCM把跟云服务商相关的东西都封装到了一个点上。下图给出了增加CCM后的结构：

![post-ccm-arch](img/post-ccm-arch.png)

## CCM组件

CCM打破了一些原本存在于KubernetesControllerMananger（KCM）上的功能，特别是KCM中原本跟云服务商产生依赖的那些控制器。这些控制器包括：

- 节点控制器（Node Controller）
- 数据卷控制器（Volume Controller）
- 路由控制器（Route Controller）
- 服务控制器（Service Controller）

在1.9版本中，CCM开始运行其中的：

- 节点控制器（Node Controller）
- 路由控制器（Route Controller）
- 服务控制器（Service Controller）

>**注意**：数据卷控制器的事儿是有意而为之的。因为之前已经做了挺多对服务商数据卷的抽象，都挺复杂的，就不打算再闹这块了。

原计划想让CCM用[Flex]()数据卷来支持可插拔的数据卷来着。但是突然出现了要准备替代Flex的[CSI]()。

所以我们打算等等CSI。

## CCM的功能

CCM继承了k8s中原来依赖于云服务商的那些组件。本节按这些组件分别进行介绍。

### 1.KubernetesControllerManager

CCM的大部分功能都来自KCM。上文书中我们说过，CCM运行以下三个控制器：

- 节点控制器（Node Controller）
- 路由控制器（Route Controller）
- 服务控制器（Service Controller）

#### 节点控制器

节点控制器要从云服务商那边获取集群节点的信息，然后对节点进行初始化。包括以下工作：

1.初始化节点时增加云服务商相关的可用区/地域标签。
2.初始化节点时增加云服务商相关的实例详情，比如类型和大小。
3.获取节点的网络地址和主机名。
4.如果节点突然放飞自我了，要从云服务商那边确认一下，看看节点是不是被删了。如果云服务商那边说节点被删了，那这边就要把k8s的节点对象也删了。

#### 路由控制器

这玩意是在云上配置正确的路由信息，让集群中不同节点上的容器可以互联互通。路由控制器只能用在GCE（Google Compute Engine）集群上。

#### 服务控制器

服务控制器要监听服务的创建、更新和删除事件。要把当前k8s的服务状态配置到云负载均衡器上（ELB、Google LB或者Oracle Infrastructure LB）。还要让这些服务保持最新状态。

### 2.Kubelet

节点控制器包含了kubelet中依赖云服务商的相关功能。CCM出现之前，kubelet要基于云服务商的信息来初始化节点，比如IP和区域/可用区标签，还有实例的类型信息等。有了CCM之后，这些事儿都交给CCM了。

在新的模型下，kubelet初始化的节点中并没有云服务商相关的信息。它在刚初始化的时候会增加一个冷屁股，关闭节点的调度功能，直到CCM为节点补充完这些信息，然后再把冷屁股去掉。

## 插件化

CCM实现使用了Go的借口，允许任意的云服务商对其进行实现然后插入功能。具体来说，就是[CloudProvider](https://github.com/kubernetes/cloud-provider/blob/9b77dc1c384685cb732b3025ed5689dd597a5971/cloud.go#L42-L62)接口。

上面提到的几个共享的控制器，它们的实现，以及一些脚手架代码，都位于k8s的核心中。和云服务商相关的实现，在核心以外进行构建，并实现了核心中定义的各种接口。

关于插件开发，详见[CCM开发]()。

## 授权

本节列出了CCOM要访问的各种API对象和相关操作。

### 节点控制器

节点控制器就是闹节点的。对于节点对象，要有完全的get、list、create、update、patch、watch和delete访问权限。

v1/Node：

- Get
- List
- Create
- Update
- Patch
- Watch
- Delete

### 路由控制器

路由控制器监听节点对象的创建，并配置正确的路由信息。需要对节点对象的get权限。

v1/Node：

- Get

### 服务控制器

服务控制器监听服务对象的create、update和delete事件，为服务配置正确的接口信息。

要访问服务，就需要list、watch权限。要更新服务，就需要update和patch权限。

要为服务创建接口信息，需要create、list、get、watch和update权限。

v1/Service：

- List
- Get
- Watch
- Patch
- Update

### 其他

CCM的实现中需要访问create事件，需要相关的安全保障，需要创建服务账户（ServiceAccount）。

v1/Event：

- Create
- Patch
- Update

v1/ServiceAccount：

- Create

CCM的RBAC ClusterRole示例：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
  - get
  - list
  - watch
  - update
```

## 云服务商实现

下面这些云服务商实现了CCM：

- [阿里云](https://github.com/kubernetes/cloud-provider-alibaba-cloud)
- [AWS](https://github.com/kubernetes/cloud-provider-aws)
- [Azure](https://github.com/kubernetes-sigs/cloud-provider-azure)
- [百度云](https://github.com/kubernetes-sigs/cloud-provider-baiducloud)
- [DigitalOcean](https://github.com/digitalocean/digitalocean-cloud-controller-manager)
- [GCP](https://github.com/kubernetes/cloud-provider-gcp)
- [Hetzner](https://github.com/hetznercloud/hcloud-cloud-controller-manager)
- [Linode](https://github.com/linode/linode-cloud-controller-manager)
- [OpenStack](https://github.com/kubernetes/cloud-provider-openstack)
- [Oracle](https://github.com/oracle/oci-cloud-controller-manager)
- [腾讯云](https://github.com/TencentCloud/tencentcloud-cloud-controller-manager)

## 集群管理

CCM完整的配置指令在[这里]()。