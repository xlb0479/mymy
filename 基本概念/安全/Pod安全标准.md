# Pod安全标准

Pod的安全设置通常是由[安全上下文](https://v1-18.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/)来定义的。安全上下文（Security Context）可以基于每个Pod来定义权限和访问控制。

此前，我们已经通过[Pod安全策略](../策略/Pod安全策略.md)实现了集群对于安全上下文的基于策略的定义方法。*Pod安全策略*属于集群级别的资源，控制着Pod定义中对安全问题敏感的部分。

但是长江后浪推前浪，一代更比一代浪。又出现了很多其他方法，增强了，甚至可以替代PodSecurityPolicy。本文主要给出在Pod安全方面的细节上的一些建议，不涉及具体的实现方式。

## 策略类型

我们非常需要一个基本的策略定义，迅速覆盖掉安全相关的场景。它们的范围从高度受限到高度灵活：

- ***Privileged***——安全不受限的策略，提供最大权限。这种策略允许提权。
- ***Baseline/Default***——做了基本的限制，阻止已知的提权问题。允许默认（最小化定义）的Pod配置。
- ***Restricted***——超级受限的策略，遵循了经历九九八十一难研制出来的Pod最佳实践。

## 策略

### Privileged

特权（Privileged）策略是非常开放的，完全不受限的。这种策略一般是提供给系统——以及基础设施——层面的工作组件，由拥有特权且授信的用户来管理。

特权策略的定义就是不加任何限制。对于默认啥都能干的场景（比如gatekeeper），特权策略不加任何约束，没啥用。相反，对于默认啥都不能干的场景（比如PodSecurityPolicy），特权策略要打开所有控制权限（关闭所有限制）。

### Baseline/Default

这种策略是为了便于常见的容器化应用来使用，避免了已知的提权风险。该策略主要提供给那些非核心应用的操作者和开发者。下面给出的控制权限应当都予以保证或拒绝。

**控制权限**|**策略**
-|-
主机的Namespace|不能允许共享主机的namespace。<br/><br/>**受限的字段：**<br/>spec.hostNetwork<br/>spec.hostPID<br/>spec.hostIPC<br/><br/>**允许值：** false
特权容器|特权容器关闭了很多安全机制，不能让这种孽畜出来祸祸。<br/><br/>**受限的字段：**<br/>spec.containers[*].securityContext.privileged<br/>spec.initContainers[*].securityContext.privileged<br/><br/>**允许值：** false、undefined/nil