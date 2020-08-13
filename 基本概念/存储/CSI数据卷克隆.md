# CSI数据卷克隆

本文介绍克隆k8s中的CSI数据卷。需要已经完成[数据卷](数据卷.md)的学习才能过来，不然你就去学学数据卷，再不然你吃个泡面去。

## 介绍

[CSI](数据卷.md#CSI)数据卷克隆可以将已存在的[PVC](持久卷（Persistent%20Volume）.md)设置到`dataSource`字段中，表示我要克隆一个[数据卷](数据卷.md)。

一个克隆（Clone）就是一个k8s数据卷的副本，可以跟普通的数据卷一样用。唯一的区别就是它来自于分配，而不是创建一个“新的”空数据卷，后端设备会创建一个指定数据卷的完全一样的副本出来。

从k8s的API的角度来看克隆的内部实现，只不过就是在创建新的PVC的时候，可以指定一个已存在的PVC作为数据源。源PVC必须已经绑定、可用（但没有正在被用）。

用户在使用这些功能的时候需要知道：

- 只有CSI驱动才支持克隆（`VolumePVCDataSource`）。
- 只有动态分配才支持克隆。
- CSI驱动不一定都实现了数据卷克隆的功能。
- 要克隆的PVC必须和当前PVC在同一个命名空间下（源和目标在同一个命名空间）。
- 只有StorageClass相同才可以克隆。
  - 目标数据卷必须和源数据卷的StorageClass相同。
  - 可以使用默认StorageClass，定义中不填storageClassName就好。
- 克隆必须发生在两个VolumeMode相同的数据卷上（如果你要申请一个块模式的数据卷，那么源数据卷必须也得是块模式）

## 分配

克隆和其他PVC一样进行分配，不同的时候需要添加一个数据源，引用同一个命名空间下的一个已存在的PVC。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: clone-of-pvc-1
    namespace: myns
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cloning
  resources:
    requests:
      storage: 5Gi
  dataSource:
    kind: PersistentVolumeClaim
    name: pvc-1
```

>**注意**：必须要设置`spec.resources.requests.storage`，而且这个值必须要大于等于源数据卷的大小。

结果就是这里出现了一个名为`clone-of-pvc-1`的新的PVC，它的内容和`pvc-1`是完全一样的。

## 使用

当新的PVC变为可用状态后，这个被克隆的PVC就跟其他PVC一样可以正常使用了。此时这个新创建的PVC是一个独立的对象。它可以被使用、克隆、做快照或者独立地被删掉，完全不用理会源PVC。也就是说源PVC和新的PVC之间不存在任何的连接，源PVC也可以正常的修改或删除，不会影响到新建的PVC。