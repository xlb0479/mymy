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

### 访问模式

### 回收策略

## 用Claim做数据卷

## 裸设备数据卷