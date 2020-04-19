# 运行时Class

**功能状态**：`Kubernetes v1.14`(beta)

本文介绍RuntimeClass资源以及运行时选择机制。

RuntimeClass是一种用来选择容器运行时的功能。容器运行时是用来运行Pod的容器的。

- [动机](#动机)
- [安装](#安装)
- [用法](#用法)
- [调度](#调度)
- [接下来……](#接下来)

## 动机

可以为每个Pod提供不同的RuntimeClass，在性能和安全性之间做出平衡。比如你要运行的东西需要更高级的安全保障，那你可以将这些Pod调度到那种能使用硬件虚拟化的容器运行时上。此时你还能获得额外的隔离性，代价就是性能上可能会有所损失。

使用了RuntimeClass后，还可以让不同的Pod运行在相同的容器运行时上，但是又拥有不一样的配置。

## 安装

确认RuntimeClass特性门已经打开（默认就开了）。如何打开可以看[特性门]()。`RuntimeClass`特性门必须在apiserver和kubelet上都打开。

1.在节点上配置CRI实现（与运行时相关）

2.创建相应的RuntimeClass资源

### 1.在节点上配置CRI实现

用RuntimeClass配置的东西都需要依赖容器运行时接口（Container Runtime Interface，CRI）。每种CRI实现都有对应的配置文档（[下面](#CRI配置)）。

>**注意**：RuntimeClass默认假设集群中所有节点的配置是同构的（即所有节点关于容器运行时的配置都是一样的）。如果要支持异构配置，看下面的[调度](#调度)部分。

每个配置都有个`handler`名称，被RuntimeClass引用。Handler必须是一个有效的DNS 1123标签（字母数字和`-`字符组成）。

### 2.创建相应的RuntimeClass资源

上一步的配置中，每个配置都要有个`handler`名称，用来标识配置。对于每个handler，创建一个对应的RuntimeClass对象。

目前的RuntimeClass资源只有2个重要的字段：名字（`metadata.name`）和handler（`handler`）。如下：

```yaml
apiVersion: node.k8s.io/v1beta1  # RuntimeClass is defined in the node.k8s.io API group
kind: RuntimeClass
metadata:
  name: myclass  # The name the RuntimeClass will be referenced by
  # RuntimeClass is a non-namespaced resource
handler: myconfiguration  # The name of the corresponding CRI configuration
```

RuntimeClass的对象名必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

>**注意**：建议将RuntimeClass的写权限（create/update/patch/delete）只授予集群管理员。这也是默认的配置。见[授权]()。

## 用法

RuntimeClass配好了之后，用起来是很简单的。在Pod的spec中指定一个`runtimeClassName`。比如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```

这样，Kubelet就会用指定的RuntimeClass来运行这个Pod。如果对应的RuntimeClass不存在，或者CRI无法运行对应的handler，Pod就会停在`Failed`[阶段]()。可以通过相关的[事件]()来查看错误信息。

如果不指定`runtimeClassName`，就会使用默认的RuntimeHandler，等效于RuntimeClass特性被关闭的情况。

### CRI配置

关于CRI配置的详细信息可以看[CRI安装]()。

#### dockershim

k8s内置的dockershim CRI不支持运行时handler。

#### containerd

通过containerd的配置文件`/etc/containerd/config.toml`来配置运行时handler。配置到runtimes一节中：

```text
[plugins.cri.containerd.runtimes.${HANDLER_NAME}]
```

详见containerd的配置文档：   
https://github.com/containerd/cri/blob/master/docs/config.md

#### CRI-O

通过CRI-O的配置文件`/etc/crio/crio.conf`来配置运行时handler。有效的Handler会配置到[crio.runtime table](https://github.com/cri-o/cri-o/blob/master/docs/crio.conf.5.md#crioruntime-table)下：

```text
[crio.runtime.runtimes.${HANDLER_NAME}]
  runtime_path = "${PATH_TO_BINARY}"
```

详见CRI-O的[配置文档](https://raw.githubusercontent.com/cri-o/cri-o/9f11d1d/docs/crio.conf.5.md)。

## 调度

**功能状态**：`Kubernetes v1.16`(beta)

从1.16版本开始，RuntimeClass加入了对异构集群的支持，通过`shceduling`字段实现。通过使用这个字段，可以确保Pod被调度到支持指定RuntimeClass的节点上。要开启调度相关的支持，必须要打开[RuntimeClass Admission控制器]()（从1.16开始就是默认打开的）。

要想确保Pod能落到支持指定RuntimeClass的节点上，对应的节点必须要有指定的标签，被`runtimeclass.scheduling.nodeSelector`选中。RuntimeClass的节点选择器会跟Pod的节点选择器合并，取二者的交集。如果有冲突，则拒绝调度Pod。

如果可选的节点设置了冷屁股，拒绝运行其他类型的RuntimeClass，你可以给RuntimeClasss设置`tolerations`。同`nodeSelector`一样，`tolerations`也会合并，取二者的并集。

要想了解更多关于配置节点选择器和热脸（tolerations）的知识，可以看看[将Pod分配到指定的节点]()。

### Pod Overhead

**功能状态**：`Kubernetes 1.18`(beta)

可以为Pod设置overhead资源。声明overhead之后，集群在制定Pod和资源相关的决策时，会考虑到你设置的这些overhead。   
要想使用Pod的overhead，必须打开PodOverhead[特性门]()（默认开启）。

在RuntimeClass的`overhead`中定义Pod的overhead。通过使用这些字段，可以为Pod使用RuntimeClass指定overhead，并且会确保这些overhead也会被k8s所参考。

## 接下来……

- [RuntimeClass设计细节](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/runtime-class.md)
- [RuntimeClass调度设计细节](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/runtime-class-scheduling.md)
- [什么是Pod Overhead]()
- [PodOverhead设计细节](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190226-pod-overhead.md)