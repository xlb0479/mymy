# Pod损耗

**功能状态**：`Kubernetes v1.18 [beta]`

当一个Pod运行在某个节点上的时候，Pod本身也是要占用一些系统资源的。这些资源不包含在Pod中容器所占用的资源内。*Pod损耗（Pod Overhead）*是来度量Pod在容器层面之上，在Pod这一层的资源使用情况。

在k8s中，Pod的损耗是在[准入](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)阶段根据Pod的[RuntimeClass](../容器/运行时Class.md)中的损耗来设定的。

当开启了Pod损耗后，在调度Pod的时候，除了容器的资源请求值，还要加上这部分损耗值。同样，kubelet在调整cgroup的时候也会考虑到这部分的损耗值，以及准备剔除Pod并对Pod进行资源占用统计排名的时候也是如此。

## 开启Pod损耗

你要确保集群上开启了`PodOverhead`[特性门](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)（在1.18的时候已经是默认开启了），还有就是在定义`overhead`字段的时候使用了某个`RuntimeClass`。

## 栗子

要使用PodOverhead功能，你需要一个定义了`overhead`字段的RuntimeClass。比如你可以使用下面的RuntimeClass定义，一个虚拟的容器运行时，每个Pod需要占用虚拟机和宿主OS大约120MiB内存：

```yaml
---
kind: RuntimeClass
apiVersion: node.k8s.io/v1beta1
metadata:
    name: kata-fc
handler: kata-fc
overhead:
    podFixed:
        memory: "120Mi"
        cpu: "250m"
```

如果创建对象的时候将RuntimeClass指定为`kata-fc`，那它的内存和CPU损耗就要计入到资源配额、节点调用，以及Pod的cgroup大小设定。

比如运行一个下面这样的对象，test-pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  runtimeClassName: kata-fc
  containers:
  - name: busybox-ctr
    image: busybox
    stdin: true
    tty: true
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
  - name: nginx-ctr
    image: nginx
    resources:
      limits:
        cpu: 1500m
        memory: 100Mi
```

在准入阶段，RuntimeClass[准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)会更新对象的PodSpec，加入在RuntimeClass中设定的`overhead`。如果PodSpec中已经定义了这个字段，Pod会被拒绝。在上面的栗子中，只指定了RuntimeClass的名字，准入控制器要修改Pod并加入`overhead`。

在RuntimeClass准入控制器操作完之后，可以看一下更新后的PodSpec：

```shell script
kubectl get pod test-pod -o jsonpath='{.spec.overhead}'
```

输出如下：

```text
map[cpu:250m memory:120Mi]
```

如果定义了ResourceQuota，容器请求值的总和还要加上`overhead`值。

当调度器判断一个Pod应该发配到哪个节点时，会同时考虑Pod的`overhead`值和容器的请求值总和。对于上面这个栗子，调度器将请求值和损耗值加到一起，然后找到一个能够提供2.25CPU和320MiB内存的节点。

一旦Pod被发配到某个节点，这个节点上的kubelet会给这个Pod创建一个新的[cgroup](https://kubernetes.io/docs/reference/glossary/?all=true#term-cgroup)。底层的容器运行时要在这个Pod中创建容器。

如果每个容器还定义了资源上限（Guaranteed QoS或Bustrable QoS，同时还有资源上限），kubelet会给对应资源设置cgroup的上限值（CPU对应cpu.cfs_quota_us，内存对应memory.limit_in_bytes）。这个上限值就是每个容器的上限值总和加上PodSpec中定义的`overhead`。

对于CPU来说，如果Pod的QoS是Guaranteed或者Burstable，kubelet会根据容器请求值的总和，加上PodSpec中的`overhead`来设定`cpu.shares`。

来看下我们的栗子，检查一下容器的请求值：

```shell script
kubectl get pod test-pod -o jsonpath='{.spec.containers[*].resources.limits}'
```

总计请求值为2000m的CPU和200MiB的内存：

```text
map[cpu: 500m memory:100Mi] map[cpu:1500m memory:100Mi]
```

然后再跟节点上看到的值来对比一下：

```shell script
kubectl describe node | grep test-pod -B2
```

输出结果显示请求了2250m的CPU和320MiB的内存，其中包含了Pod损耗：

```text
  Namespace                   Name                CPU Requests  CPU Limits   Memory Requests  Memory Limits  AGE
  ---------                   ----                ------------  ----------   ---------------  -------------  ---
  default                     test-pod            2250m (56%)   2250m (56%)  320Mi (1%)       320Mi (1%)     36m
```

## 检查Pod的cgroup限制

当组件正在运行的时候，检查一下Pod在节点上的内存cgroup。在下面的栗子中，我们在节点上使用了[`crictl`](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)，这是一个兼容CRI容器运行时的命令行工具。这个栗子中用来显示Pod损耗的方式是比较高阶的，一般人可学不会，正常来说，用户不需要去节点上直接检查cgroup。

首先，选择一个节点，获取Pod的标识：

```shell script
# Run this on the node where the Pod is scheduled
POD_ID="$(sudo crictl pods --name test-pod -q)"
```

然后你就可以获取Pod的cgroup路径：

```shell script
# Run this on the node where the Pod is scheduled
sudo crictl inspectp -o=json $POD_ID | grep cgroupsPath
```

输出的cgroup路径包含了Pod的`pause`容器。Pod级别的cgroup是在上一级目录中。

```text
        "cgroupsPath": "/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/7ccf55aee35dd16aca4189c952d83487297f3cd760f1bbf09620e206e7d0c27a"
```

此时看到，Pod的cgroup路径是`kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2`。来检查一下Pod级别的内存cgroup设置：

```shell script
# Run this on the node where the Pod is scheduled.
# Also, change the name of the cgroup to match the cgroup allocated for your pod.
 cat /sys/fs/cgroup/memory/kubepods/podd7f4b509-cf94-4951-9417-d1087c92a5b2/memory.limit_in_bytes
```

结果是320MiB，跟我们想的一样：

```text
335544320
```

### 观测性

在[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)中有一个`kube_pod_overhead`指标，可以用来标识是否用到了Pod损耗，而且还能用它来观察定义了损耗值的Pod是否能够稳定运行。这个功能在kube-state-metrics的1.9 release版本中还不可用，希望在接下来的版本中能用到它。目前，用户需要手动用kube-state-metrics源码进行编译才行。

## 接下来……

- [RuntimeClass](../容器/运行时Class.md)
- [PodOverhead设计思想](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190226-pod-overhead.md)