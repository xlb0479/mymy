# 持久卷（Persistent Volumes）

本文介绍了当前版本k8s中的*持久卷（persistent volumes）*。建议你先了解什么是[数据卷](数据卷.md)。

## 介绍

管理存储不同于管理计算实例。持久卷系统为用户和管理者提供了一套API，对存储的提供和使用细节进行了抽象。要实现这些，我们引入两个新的API资源：持久卷和持久卷Claim（PersistentVolumeClaim）。

一个*持久卷*（PV）就是集群中的一块存储，由管理员分配，或使用[Storage Class](Storage%20Class.md)进行动态分配。它就是集群中的一个资源，就像节点一样，都是属于集群的资源。PV是数据卷插件，类似数据卷，但是PV的生命周期不依赖于使用它的Pod。这种API对象包含了存储的实现细节，比如NFS、iSCSI，或者特定云服务商的存储系统。

一个*持久卷Claim（PersistentVolumeClaim，PVC）*是用户发起的一个存储请求。它跟Pod类似。Pod消耗的是节点资源，而PVC消耗的是PV资源。Pod可以请求特定层次的资源（CPU和内存）。Claim可以请求特定大小和访问模式（比如ReadWriteOnce、ReadOnlyMany或者ReadWriteMany，见[访问模式](#访问模式)）。

PVC可以让用户使用抽象存储资源，用户对于不同的问题，比如性能方面，使用PV的时候会选择不同的属性。集群管理者应当提供各种不同的PV，不仅仅是大小和访问模式的不同，而且不需要将这些数据卷的实现细节暴露给用户。对于这件事情，就需要用到*StorageClass*资源了。

见[配置Pod和PV](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)。

## 数据卷和claim的生命周期

PV是集群的资源。PVC是对这些资源的请求，它同样还可以对这些资源进行检查。PV和PVC的交互具备如下生命周期：

### 分配

可以用两种方式来分配PV：静态的和动态的。

#### 静态

集群管理员创建一堆PV。它们包含了存储的具体细节，提供给集群用户使用。它们都已经存储于k8s的API中了。

#### 动态

如果管理员创建的静态PV无法匹配用户的PVC，集群就要专门为这个PVC动态分配一个数据卷了。此时的分配要基于StorageClass：PVC必须要请求一个[StorageClass](Storage%20Class.md)，集群管理员必须要创建并配置好这个Class才能进行动态分配。如果Claim请求的Class是“”，相当于主动关闭动态分配。

如果要基于StorageClass来开启动态存储分配，集群管理员需要开启api server的`DefaultStorageClass`[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)。可以检查一下api server组件的`--enable-admission-plugins`选项中是否包含`DefaultStorageClass`。关于api server的命令行选项，可以去看[kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)文档。

### 绑定

一个用户创建，或者是动态分配时已经创建，一个PVC，包含了指定数量的存储以及某种访问模式。master中会有一个控制循环不断监视新的PVC，寻找匹配的PV（如果有），然后将它们绑定起来。如果为新的PVC动态地分配了一个PV，控制循环直接就把这个PV绑定到PVC上。否则用户总是能得到至少满足需求的资源，但是数据卷可能要比申请的要大、要多。一旦绑定了，PVC的绑定是独占的，不管它们是怎么绑定的。一个PVC到PV的绑定是一对一的，使用了一个ClaimRef，对PV和PVC进行双向绑定。

若果找不到匹配的数据卷，Claim就一直处于未绑定的状态。Claim直到出现匹配的数据卷才会绑定。比如一个集群分配了大量50Gi的PV，但是不会匹配一个100Gi的PVC请求。直到一个100Gi的PV加到集群中，这个PVC才能绑定。

### 使用

Pod将Claim作为数据卷。集群会探知到Claim绑定的数据卷，然后将数据卷挂载到Pod中。对于支持多种访问模式的数据卷，用户在Pod中通过Claim使用数据卷的时候指定他们所需的访问模式。

当用户拥有一个Claim，Claim也绑定好了，只要用户需要，绑定的PV就一直属于用户。用户调度他们的Pod，在Pod的`volumes`块中包含一个`persistentVolumeClaim`来访问他们申请到的PV。详见[用Claim做数据卷](#用Claim做数据卷)。

### 保护正在使用的存储对象

这种特性是为了确保正在被Pod使用的PVC，一级绑定到这些PVC上的PV，不会被操作系统删除，不然的话数据就丢失了，就全完犊子了。

>**注意**：PVC被Pod使用的前提是Pod对象存在，并且正在使用这个PVC。

如果用户删除了一个正在被Pod使用的PVC，PVC不会立即被删除。PVC的删除会一直延期，知道这个PVC不在被任何Pod使用。还有，如果一个管理员删除了一个已经绑定到PVC的PV，这个PV也不会立即被删除。PV的删除会一直延期，直到PV不再绑定到任何PVC上。

你会看到当PVC的状态是`Terminating`时，PVC是受保护的，`Finalizers`列表中包含了`kubernetes.io/pvc-protection`：

```text
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

你会看当PV的状态是`Terminating`时，PV是受保护的，`Finalizers`列表中同样包含了`kubernetes.io/pv-protection`：

```text
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```

### 回收（Reclaiming）

当用户用完了他们的数据卷，可以用API删除这些PVC对象，触发资源回收。PV的回收策略告诉集群当PV从它的Claim中释放出来之后要做什么。目前来说是，数据卷可以保留（Retain）、回收（Recycle）或删除（Delete）。

#### 保留（Retain）

`Retain`这种回收策略可以允许手动对资源进行回收。当PVC删除后，PV一直存在，数据卷被认为是得到了“释放”。但此时还不能提供给另一个Claim使用，因为前一个Claim的数据还留在数据卷中。管理员可以按照以下步骤手动回收数据卷。

- 1.删除PV。关联在外部设备（比如AWS EBS、GCE PD、Azure Disk或者Cinder数据卷）中的存储资产在PV删除后依然能够一直保留。
- 2.手动删除存储资产上的数据。
- 3.手动删除存储资产，或者如果你想重用同样的存储资产，就用它们创建一个新的PV。

#### 删除（Delete）

如果数据卷插件支持`Delete`这种回收策略，删除操作会从k8s中删除PV对象，以及管理在外部设备，比如AWS EBS、GCE PD、Azure Disk或者Cinder数据卷，中的存储资产。动态分配的数据卷会继承[StorageClass的回收策略]()，默认就是`Delete`。管理员应当根据用户的期望来配置StorageClass；否则PV必须要在创建候进行编辑（edit）或者打补丁（patch）了。见[修改一个PV的回收策略](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/)。

#### 回收（Recycle）

>**警告**：`Recycle`这种回收策略已经废弃了。作为替代，建议使用动态分配。

如果底层的数据卷插件能支持，`Recycle`这种回收策略会对数据卷执行一个简单的清理（`rm -rf /thevolume/*`），然后提供给新的Claim。

但是呢，管理员其实可以用k8s的controller manager的命令行参数配置一个自定义的回收器Pod，详见[参考手册](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)。这种自定义的回收器Pod必须包含一个`volumes`声明，如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "k8s.gcr.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

但是，这个自定义的回收器Pod中的`volumes`的路径部分部分会被替换成要回收的数据卷的路径。（这个不知道是k8s会自动替换还是需要咱们手动替换，应该是要手动替换的吧，command里面不也得改么）

### PVC扩容

**功能状态**：`Kubernetes v1.11 [beta]`

现在默认就开启了对PVC扩容的支持。你可以扩容以下类型的数据卷：

- gcePersistentDisk
- awsElasticBlockStore
- Cinder
- glusterfs
- rbd
- Azure File
- Azure Disk
- Portworx
- FlexVolumes
- CSI

只有当PVC的StorageClass的`allowVolumeExpansion`设置为true的情况下才可以对其进行扩容。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

要想申请一个更大的PVC，修改PVC对象然后指定一个更大的容量。这就会触发数据卷的扩容，进而传递到底层PV上。不会重建一个新的PV来满足Claim，而是对当前数据卷进行扩容。

#### CSI数据卷扩容

**功能状态**：`Kubernetes v1.16 [beta]`

CSI数据卷默认就可以支持扩容，但是需要对应的CSI驱动也必须能够支持数据卷扩容。参考特定的CSI驱动文档了解更多信息。

#### 对包含文件系统的数据卷进行大小调整

如果数据卷包含了一个文件系统，只有是XFS、Ext3或Ext4的时候才能对数据卷进行大小调整。

如果数据卷包含了文件系统，只有在新的Pod用`ReadWrite`模式使用这个PVC时才会被调整大小。文件系统扩容只有在Pod启动，或者说在Pod运行的时候但是底层文件系统支持在线扩容的情况下执行。

FlexVolume可以在驱动的`RequiresFSResize`设置为`true`的情况下允许调整大小。FlexVolume可以在Pod重启的时候进行大小调整。

#### 对正在使用的PVC调整大小

**功能状态**：`Kubernetes v1.15 [beta]`

>**注意**：从1.15版本开始，对使用中的PVC进行扩容进入到了beta版本，从1.11开始一直是alpha版本。必须要开启`ExpandInUsePersistentVolumes`特性，很多集群对beta特性默认都是自动开启的。参见[特性门](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)了解更多内容。

在这种情况下，不需要删除或重建那些正在使用PVC的Pod或Deployment。当正在使用的PVC的文件系统被扩容时，PVC对Pod一直保持可用状态。对于没有被Pod或Deployment使用的PVC，这种特性没有屁用。在扩容完成前，你必须创建一个使用该PVC的Pod。（最后这个倒是没想到）

和其他数据卷类型差不多——FlexVolume数据卷也可以在被Pod使用的时候进行扩容。

>**注意**：FlexVolume要想支持调整大小，底层的驱动必须也同时支持。

>**注意**：对EBS数据卷进行扩容可是个耗时的活儿。而且，每个数据卷有一个配额，每6个小时允许修改一次。

#### 数据卷扩容时进行错误恢复

如果扩容底层存储出错，集群管理员可以手动恢复PVC的状态，并取消调整大小的请求。否则，如果没有管理员介入，控制器会不断地重试该请求。

- 1.将绑定到PVC的PV的回收策略标记为`Retain`。
- 2.删除PVC。因为PV的回收策略是`Retain`——重建PVC不会丢失任何数据。
- 3.删除PV的`claimRef`属性，这样新的PVC就可以绑定它了。这一步应该会让PV变为`Available`。
- 4.用一个比PV稍小的容量来重建PVC，并将PVC的`volumeName`字段设置为PV的的名字。这一步可以让新的PVC绑定到已有的PV上。
- 5.不要忘记恢复PV的回收策略。

## PV的类型

PV的各种类型被实现成了插件。k8s目前支持以下插件：

- GCEPersistentDisk
- AWSElasticBlockStore
- AzureFile
- AzureDisk
- CSI
- FC (Fibre Channel)
- FlexVolume
- Flocker
- NFS
- iSCSI
- RBD (Ceph Block Device)
- CephFS
- Cinder (OpenStack block storage)
- Glusterfs
- VsphereVolume
- Quobyte Volumes
- HostPath (Single node testing only -- local storage is not supported in any way and WILL NOT WORK in a multi-node cluster)
- Portworx Volumes
- ScaleIO Volumes
- StorageOS

## PV

每个PV包含一个spec和一个status，也就是数据卷的定义和状态。PV对象的名字必须是有效的[DSN子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

>**注意**：在集群中使用一个PV的时候可能需要对应数据卷类型的辅助程序。在本例中，这个PV是NFS类型的，需要有/sbin/mount.nfs这个辅助程序来支持NFS文件系统的挂载。

### 容量

一般来说一个PV都有一个特定的存储容量。是通过PV的`capacity`属性来设置的。可以去看k8s的[资源模型](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/resources.md)了解`capacity`的单位。

目前来看，存储大小是唯一可以被设置或被请求的资源。未来可能还会提供IOPS、吞吐量等属性。

### 数据卷模式

**功能状态**：`Kubernetes v1.18 [stable]`

k8s的PV支持两种`volumeMode`：`Filesystem`和`Block`。

`volumeMode`是个可选参数。如果不填，默认是`Filesystem`。

带有`volumeMode`的数据卷：`Filesystem`是*被挂载*到了Pod的一个目录中。如果数据卷底层是一个块设备，并且设备是空的，k8s会在首次进行挂载前，先在这个设备上创建一个文件系统。

可以将`volumeMode`设置为`Block`，作为一个裸设备来使用数据卷。这种数据卷在Pod中表现为一个块设备，没有任何文件系统。这种模式可以为Pod提供最快速的访问数据卷的方式，在Pod和数据卷中间没有任何文件系统层。另一方面，运行在Pod中的应用必须要知道如何处理裸设备。见[裸设备数据卷支持](#裸设备数据卷)，其中包含了一个在Pod中使用`volumeMode: Block`的栗子。

### 访问模式

只要是资源提供者支持的方式，PV可以用各种方式挂载到主机上。如下表所见，提供者具有不同的能力，每个PV也设置成了特定数据卷所支持的访问模式。比如吧，NFS可以支持多个客户端的读写，但某一个NFS的PV可能只设置成只读模式。每个PV有自己的访问模式集合，描述一个特定PV的能力。（我觉得我快成机器翻译了）

访问模式有：

- ReadWriteOnce——数据卷可以被一个单独的节点以读写方式挂载。
- ReadOnlyMany——数据卷可以被多个节点以只读的方式挂载。
- ReadWriteMany——数据卷可以被多个节点以读写的方式挂载。

在命令行中，访问模式简写成：

- RWO——ReadWriteOnce
- ROX——ReadOnlyMany
- RWX——ReadWriteMany

>**重要！**一个数据卷一次只能使用一种访问模式，即便它支持多种也不行。比如吧，一个GCEPersistentDisk可以被一个单独的节点挂载成ReadWriteOnce，或者被多个节点挂载成ReadOnlyMany，但是！但是不能同时存在。

数据卷插件|ReadWriteOnce|ReadOnlyMany|ReadWriteMany
-|-|-|-
AWSElasticBlockStore|✓|-|-
AzureFile|✓|✓|✓
AzureDisk|✓|-|-
CephFS|✓|✓|✓
Cinder|✓|-|-
CSI|依赖于驱动|依赖于驱动|依赖于驱动
FC|✓|✓|-
FlexVolume|✓|✓|依赖于驱动
Flocker|✓|-|-
GCEPersistentDisk|✓|✓|-
Glusterfs|✓|✓|✓
HostPath|✓|-|-
iSCSI|✓|✓|-
Quobyte|✓|✓|✓
NFS|✓|✓|✓
RBD|✓|✓|-
VsphereVolume|✓|-|- (Pod在一起的时候可以)
PortworxVolume|✓|-|✓
ScaleIO|✓|✓|-
StorageOS|✓|-|-

### 每个PV有一个Class，也就是`storageClassName`，对应一个[StorageClass](Storage%20Class.md)的名字。一个指定Class的PV只能绑定到请求该Class的PVC上。一个没有`storageClassName`的PV就没有Class，只能绑定到没有指定Class的PVC上。

在老老年间，我们用的是`volume.beta.kubernetes.io/storage-class`注解而不是`storageClassName`属性。现在这个注解依然管用；但是在未来的版本中会被完全废弃。

### 回收策略

当前的回收策略有：

- Retain——手动回收
- Recycle——基本清理（`rm -rf /thevolume/*`）
- Delete——相关的存储资产，比如AWS EBS、GCE PD、Azure Disk或者OpenStack Cinder，数据卷会被删除。

目前只有NFS和HostPath支持Recycle。AWS EBS、GCE PD、Azure Disk以及Cinder数据卷支持Delete。

### 挂载选项

当一个PV挂载到一个节点上时，k8s管理员可以指定额外的挂载选项。

>**注意**：不是所有的PV都支持挂载选项。

一下数据卷类型支持挂载选项：

-AWSElasticBlockStore
-AzureDisk
-AzureFile
-CephFS
-Cinder (OpenStack block storage)
-GCEPersistentDisk
-Glusterfs
-NFS
-Quobyte Volumes
-RBD (Ceph Block Device)
-StorageOS
-VsphereVolume
-iSCSI

不会对挂载选项进行校验，如果某个选项不对，挂载直接失败。

在老老年间，我们使用`volume.beta.kubernetes.io/mount-options`而不是`mountOptions`属性。现在这个属性依然管用；但是，在未来的版本中会被完全废弃。

### 节点亲和性（Affinity）

>**注意**：对于大部分数据卷类型，你不需要设置这个字段。它会自动注入到[AWS EBS]()、[GCE PD]()以及[Azure Disk]()数据卷类型中。对于[local](数据卷.md#local)数据卷你才需要进行明确设置。

一个PV可以通过设置[节点亲和性](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#volumenodeaffinity-v1-core)来定义约束，限制该数据卷可以被哪些节点访问。使用该PV的Pod，只会被调度到那些被节点亲和性选中的节点上。

### 阶段（Phase）

一个数据卷会处于以下阶段之一：

- Available——空闲资源，尚未绑定到任何Claim
- Bound——已经绑定到某个Claim上了
- Released——它的Claim已经被删除了，但是资源尚未被集群回收
- Failed——数据卷自动回收失败

命令行中可以显示出PV绑定的PVC的名字。

### PVC

每个PVC包含一个spec和一个status，也就是它的定义和状态。PVC对象的名字必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### 访问模式

对存储请求特定访问模式的时候，Claim和数据卷保持同样的约定。

### 数据卷模式

也是一样，跟数据卷保持同样的约定，指定数据卷是一个文件系统还是块设备。

### 资源

Claim跟Pod差不多，可以请求指定数量的资源。本例中请求的是存储资源。同样的[资源模型](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/resources.md)适用于数据卷和Claim。

### 选择器

Claim可以定义一个[标签选择器](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)，对数据卷集合进行进一步的过滤。只有标签跟选择器匹配的数据卷才能绑定到Claim上面。选择器由两个字段组成：

- `matchLabels`——数据卷必须要有一个同样的标签以及同样的值
- `matchExpressions`——一个列表，包含指定的key、value列表，以及将key和value关联起来的operator。有效的operator包括In、NotIn、Exists和DoesNotExist。

所有这些要求，`matchLabels`和`matchExpressions`，是要取交集的——要想匹配必须全部满足。

### Class

通过将`storageClassName`设置成[StorageClass](Storage%20Class.md)的名字，一个Claim可以请求一个特定的Class。只有Class和被请求的Class相同的PV，名字和PVC的`storageClassName`相同，才能绑定到PVC上。

PVC不是必须要请求一个Class。如果PVC将`storageClassName`设置为`""`，会被推断为请求一个没有Class的PV，因此只能绑定到没有Class的PV上（老老年间就是没有注解，或者没有设置为`""`）。没有`storageClassName`的PVC不太一样（注意是没有，而不是设置成`""`），集群会做特殊处理，这依赖于是否开启了[`DefaultStorageClass`admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)。

- 如果开启了admission plugin，管理员要指定一个默认的StorageClass。所有不带`storageClassName`的PVC只会绑定到该默认值的PV上。可以将StorageClass对象的`storageclass.kubernetes.io/is-default-class`注解设置为`true`来将它这是为默认的StorageClass。如果管理员没有设置默认对象，那集群就会假装没有开启admission plugin。如果指定了多个默认对象，admission plugin会禁止所有PVC的创建。
- 如果关闭了admission plugin，没有默认的StorageClass标记。所有不带`storageClassName`的PVC只能绑定到不带Class的PV上，此时，不带`storageClassName`的PVC跟那些将`storageClassName`设置为`""`的PVC就一样了。

跟你的安装方式有关，可能在集群安装的时候addon manager就已经将默认的StorageClass部署到集群中了。

如果PVC请求了一个StorageClass，同时还设置了一个`selector`，这两个请求要取交集：只有跟请求的Class匹配的PV，并且标签也匹配的PV，才能绑定到PVC上。

>**注意**：目前来说，如果PVC的`selector`非空，则不会为其动态分配PV。

在老老年间，我们使用`volume.beta.kubernetes.io/storage-class`注解而不是`storageClassName`属性。这个注解现在依然管用；但是，在未来的版本中会被完全废弃。

## Claim做数据卷



## 裸设备数据卷