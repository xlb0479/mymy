# Pod安全标准

【牛逼的译者注：如果你觉得之前的翻译很糟糕，那本文就让你见识一下什么叫真正的糟糕】

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
特权容器|特权容器关闭了很多安全机制，不能让这种孽畜出来祸祸。<br/><br/>**受限的字段：**<br/>spec.containers[\*].securityContext.privileged<br/>spec.initContainers[\*].securityContext.privileged<br/><br/>**允许值：** false、undefined/nil
Cap|不能允许使用除了[默认集合](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)以外的其他能力。<br/><br/>**受限的字段：**<br/>spec.containers[\*].securityContext.capabilities.add<br/>spec.initContainers[\*].securityContext.capabilities.add<br/><br/>**允许值：** empty (或者局限于某个固定的列表)
HostPath数据卷|不让用这种数据卷。<br/><br/>**受限的字段：**<br/>spec.volumes[\*].hostPath<br/><br/>**允许值：** undefined/nil
HostPort|不让用这种端口，或者局限在一个最小化的范围内。<br/><br/>**受限的字段：**<br/>spec.containers[\*].ports[*].hostPort<br/>spec.initContainers[\*].ports[\*].hostPort<br/><br/>**允许值：** 0、undefined (或者局限于某个固定的列表)
AppArmor（可选）|对于可以支持该功能的主机，默认使用的是AppArmor的“runtime/default”策略。默认策略应当能够阻止对策略的重写和关闭，或者只能允许一部分的策略被重写。<br/><br/>**受限的字段：**<br/>metadata.annotations\['container.apparmor.security.beta.kubernetes.io/*'\]<br/><br/>**允许值：** “runtime/default”, undefined
SELuinux（可选）|拒绝自定义SELinux选项。<br/><br/>**受限的字段：**<br/>spec.securityContext.seLinuxOptions<br/>>spec.containers[\*].securityContext.seLinuxOptions<br/>spec.initContainers[\*].securityContext.seLinuxOptions<br/><br/>**允许值：** undefined/nil
/proc的挂载类型|默认的/proc mask能够减少受攻击面，应当开启。<br/><br/>**受限的字段：**<br/>spec.containers[*].securityContext.procMount<br/>spec.initContainers[\*].securityContext.procMount<br/><br/>**允许值：** undefined/nil，“Default”
Sysctls|Sysctls可以关闭安全机制，或者影响到主机上的所有容器，不能允许这种功能瞎祸祸，除了一小部分“安全”的操作可以用一下。一个在容器或Pod命名空间下的sysctl被认为是安全的，它跟同一个节点上的其他Pod或者进程是互相隔离的。<br/><br/>**受限的字段：**<br/>spec.securityContext.sysctls<br/><br/>**允许值：** kernel.shm_rmid_forced<br/>net.ipv4.ip_local_port_range<br/>net.ipv4.tcp_syncookies<br/>net.ipv4.ping_group_range<br/>undefined/empty

### Restricted

这种策略就是强制Pod去遵循最佳实践，会牺牲一些兼容性。这种策略的目标群体是那些对安全问题非常敏感的应用的操作员和开发者，以及低级授信用户。下面的空值权限应当予以保证或拒绝。

**控制权限**|**策略**
-|-
*都是默认策略*
数据卷类型|除了要限制HostPath数据卷，还要限制PV中定义的非主流的数据卷类型。<br/><br/>**受限的字段：**<br/>spec.volumes[\*].hostPath<br/>spec.volumes[\*].gcePersistentDisk<br/>spec.volumes[\*].awsElasticBlockStore<br/>spec.volumes[\*].gitRepo<br/>spec.volumes[\*].nfs<br/>spec.volumes[\*].iscsi<br/>spec.volumes[\*].glusterfs<br/>spec.volumes[\*].rbd<br/>spec.volumes[\*].flexVolume<br/>spec.volumes[\*].cinder<br/>spec.volumes[\*].cephFS<br/>spec.volumes[\*].flocker<br/>spec.volumes[\*].fc<br/>spec.volumes[\*].azureFile<br/>spec.volumes[\*].vsphereVolume<br/>spec.volumes[\*].quobyte<br/>spec.volumes[\*].azureDisk<br/>spec.volumes[\*].portworxVolume<br/>spec.volumes[\*].scaleIO<br/>spec.volumes[\*].storageos<br/>spec.volumes[\*].csi<br/><br/>**允许值：** undefined/nil
提权（Privilege Escalation）|提权（比如通过set-user-ID或者set-group-ID文件）都应该被拒绝。<br/><br/>**受限的字段：**<br/>spec.containers[\*].securityContext.allowPrivilegeEscalation<br/>spec.initContainers[\*].securityContext.allowPrivilegeEscalation<br/><br/>**允许值：** false