# StorageClass

本文介绍k8s中的StorageClass。你需要已经完成[数据卷](数据卷.md)和[持久卷](持久卷（Persistent%20Volume）.md)的学习。

## 介绍

一个StorageClass可以让管理员描述他们提供的存储的“类型”。不同的类型可能代表了不同的qos，或者是备份策略，又或者是管理员自定的其他策略。k8s对于类型信息可以表达什么东西是比较开放的。这种概念在其他存储系统中可能会被称为是“profile”。

## StorageClass资源

每个StorageClass包含了`provisioner`、`parameters`和`reclaimPolicy`，使用这个类型的PV在进行动态分配的时候都需要用到这些属性。

StorageClass对象的名字非常重要，用户就是用它来指定要申请的类型。管理员创建StorageClass对象的时候要设置名字和其他参数，对象创建候就不能被删除了。

管理员可以创建一个默认的StorageClass，给那些没有指定类型的PVC用：详见[PVC](持久卷（Persistent%20Volume）.md#pvc)。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### 分配器

每个StorageClass都有一个分配器（provisioner），用来指定使用哪种数据卷插件来分配PV。这是个必填字段。

**数据卷插件**|**内部分配器**|栗子（只写我感兴趣的）
-|-|-
AWSElasticBlockStore|✓|[AWS EBS]()
AzureFile|✓|[Azure File]()
AzureDisk|✓|[Azure Disk]()
CephFS|-|-
Cinder|✓|[OpenStack Cinder]()
FC|-|-
FlexVolume|-|-
Flocker|✓|-
GCEPersistentDisk|✓|[GCE PD]()
Glusterfs|✓|[Glusterfs]()
iSCSI|-|-
Quobyte|✓|[Quobyte]()
NFS|-|-
RBD|✓|[Ceph RBD]()
VsphereVolume|✓|[vSphere]()
PortworxVolume|✓|[Portworx Volume]()
ScaleIO|✓|[ScaleIO]()
StorageOS|✓|[StorageOS]()
Local|-|[Local](#Local)

你能用的不仅限于这里列出的“内部”分配器（前缀都是“kubernetes.io”，是和k8s一起发布的）。你可以运行并指定外部的分配器，它们都是独立的程序，遵从k8s定义的[规范](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-provisioning.md)。外部分配器的作者完全控制着他们自己的代码和分配器的发版机制，以及如何运行、用了什么数据卷插件（包括Flex）等等。在[kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner)代码仓库中维护了一个用来编写外部分配器的库，实现了k8s的规范。在[kubernetes-incubator/external-storage](https://github.com/kubernetes-retired/external-storage)中列出了一些外部分配器。

比如NFS不提供内部分配器，但是可以用外部分配器。还有一些存储厂商提供了他们自己的第三方外部分配器。

### 回收策略

由StorageClass动态创建的PV的回收策略就是用了类型中的`reclaimPolicy`字段，可以使`Delete`或`Retain`。如果创建StorageClass的时候没有指定`reclaimPolicy`，默认是`Delete`。

如果是手动创建的PV，由StorageClass管理的，它们的回收策略使用的是创建时指定的回收策略。

### 允许数据卷扩容

**功能状态**：`Kubernetes v1.11 [beta]`

PV可以配置成支持扩容的。当设置成`true`后，允许用户通过编辑PVC对象来调整数据卷的大小。

以下数据卷支持数据卷扩容，可以将它们的StorageClass的`allowVolumeExpansion`设置为true。

**数据卷类型**|需要的k8s版本
-|-
gcePersistentDisk|1.11
awsElasticBlockStore|1.11
Cinder|1.11
glusterfs|1.11
rbd|1.11
Azure File|1.11
Azure Disk|1.11
Portworx|1.11
FlexVolume|1.13
CSI|1.14 (alpha), 1.16 (beta)

>**注意**：只能扩容数据卷，不能缩小。

### 挂载选项

由StorageClass动态创建的PV使用类型中`mountOptions`字段设置的挂载选项。

如果指定了挂载选项但是数据卷插件不支持挂载选项，则导致分配失败。类型和PV都不会去验证挂载选项，所以如果不对，PV挂载就直接失败。

### 数据卷绑定模式

### Local