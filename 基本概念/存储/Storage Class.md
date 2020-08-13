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

`volumeBindingMode`字段控制着[数据卷绑定和动态分配](持久卷（Persistent%20Volume）.md#分配)的时机。

默认情况下，`Immediate`模式就是说一旦PVC创建出来，就立即进行数据卷绑定和动态分配。对于有拓扑限制的存储，不是集群所有节点都能访问的那种存储，PV在绑定活分配的时候是不知道Pod的调度情况的。这就会导致Pod调度异常。

集群管理员可以通过设置`WaitForFirstConsumer`模式来指出这种情况，这样，就会延后PV的绑定和分配，直到使用PVC的Pod被创建出来。PV会根据Pod调度的拓扑约束来进行选择或分配。依据的约束条件包括但不限于[资源需求](../配置/管理容器资源.md)、[节点选择器](../调度和驱逐/将Pod指派到节点.md#nodeselector)、[Pod亲和性以及反亲和性](../调度和驱逐/将Pod指派到节点.md#亲和性与反亲和性)，还有[热脸和冷屁股](../调度和驱逐/热脸和冷屁股.md)。

以下插件支持在`WaitForFirstConsumer`模式下的动态分配（我都没翻译）：

- [AWSElasticBlockStore]()
- [GCEPersistentDisk]()
- [AzureDisk]()

以下插件支持`WaitForFirstConsumer`模式下的预创建PV的绑定：

- 以上所有
- [Local](#Local)

**功能状态**：`Kubernetes v1.17 [stable]`

[CSI数据卷](数据卷.md#CSI)可以支持动态分配和预创建的PV，但是你需要阅读特定CSI驱动的文档，看看它们支持的拓扑key和栗子。

### 允许的拓扑

当用户设置了`WaitForFirstConsumer`这种数据卷绑定模式，大部分情况下都不需要将分配设置到特定拓扑中了。当然，如果依然有需求，可以设置`allowedTopologies`。

本例演示了如何将分配数据卷的拓扑限制到特定的可用区中，可以替换插件中的`zone`和`zones`字段。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
```

## 参数

StorageClass有一些参数用来描述属于该Class的数据卷。实际可用的参数要依赖于`provisioner`。比如，`type`参数值为`io1`，以及`iopsPerGB`参数，都是EBS特有的。如果某个参数没有写，会有对应的默认值。

一个StorageClass中最多可以定义512个参数。参数对象的最大长度不能超过256KB，包括了key和value的长度。

下面只列出我感兴趣的provisioner。

### Ceph RBD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.16.153.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

- `monitors`：Ceph的monitor列表，用逗号间隔。必填项。
- `adminId`：Ceph的ClientID，需要能在pool中创建镜像。默认是“admin”。
- `adminSecretName`：`adminId`的Secret名。必填项。对应的Secret类型必须为“kubernetes.io/rbd”。
- `adminSecretNamespace`：`adminSecretName`的命名空间。默认是“default”。
- `pool`：Ceph RBD pool。默认是“rbd”。
- `userId`：

### Local