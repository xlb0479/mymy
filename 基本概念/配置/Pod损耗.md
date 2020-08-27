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