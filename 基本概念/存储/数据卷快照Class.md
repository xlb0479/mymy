# 数据卷快照Class

本文介绍k8s中的VolumeSnapshotClass。你最好已经学习过[数据卷快照](数据卷快照.md)和[StorageClass](Storage%20Class.md)再过来，不然你就搞笑呢。

## 介绍

跟StorageClass的作用类似，StorageClass是为管理员在分配一个数据卷的时候，提供一种描述存储“类型”的方法，VolumeSnapshotClass则是在分配一个数据卷快照的时候，提供一个描述存储“类型”的方式。

## VolumeSnapshotClass资源

每个VolumeSnapshotClass包含了`driver`、`deletionPolicy`和`parameters`字段，用于属于该Class的VolumeSnapshot需要进行动态分配的时候。

VolumeSnapshotClass的名字很重要，是用来让用户指定请求一个特定的类型的。管理员在首次创建VolumeSnapshotClass对象的时候就要设置名字和其他参数，对象在创建完后不能更改。

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

管理员可以为那些没有指定任何Class的VolumeSnapshot设置一个默认的VolumeSnapshotClass，也就是添加一个`snapshot.storage.kubernetes.io/is-default-class: "true"`注解：

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

### Driver

数据卷快照类型包含了一个驱动，用来决定在分配VolumeSnapshot的时候使用哪个CSI数据卷插件。必填项。

### DeletionPolicy

数据卷快照类型有一个deletionPolicy。它可以让你配置当VolumeSnapshot对象被删除的时候，绑定的VolumeSnapshotContent该怎么办。可以是`Retain`或`Delete`。必填项。

如果deletionPolicy是`Delete`，那底层存储快照就会跟VolumeSnapshotContent对象一起删除。如果是`Retain`，底层快照和VolumeSnapshotContent都能得以保留。

## Parameters

数据卷快照类型可以有参数来描述属于该类型的数据卷快照。不同的`driver`会有不同的参数。