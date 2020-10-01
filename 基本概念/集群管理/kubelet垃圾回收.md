# kubelet垃圾回收

垃圾回收（Garbage Collection）对于kubelet来说可真是太有用了，可以清理不用的镜像和容器。kubelet可以每分钟执行一次容器垃圾回收，每五分钟执行一次镜像垃圾回收。

不建议使用外部垃圾回收工具，这些工具可能会碰巧破坏kubelet的规则，删除那些本以为应该依然存在的容器。

## 镜像回收

k8s通过imageManager以及cadvisor管理所有镜像的生命周期。

镜像垃圾回收的策略包含两个要素：`HighThresholdPercent`和`LowThresholdPercent`。磁盘占用超过上限阈值便会触发垃圾回收。垃圾回收会删除最近最少使用的镜像，直到到达下限阈值。

## 容器回收

容器的回收要考虑到三个由用户设定的变量。`MinAge`是容器可以被回收的最小年龄。`MaxPerPodContainer`是每一个Pod对儿（UID和容器名）允许出现死掉容器的数量最大值。`MaxContainers`是所有死掉的容器数量最大值。三个变量都可以单独关掉，比如将`MinAge`设成零，将`MaxPerPodContainer`和`MaxContainers`都可以设置成小于零。

kubelet会瞄准那些无标识的、被删除的，或者是上面提到的限定范围之外的容器。最老的容器一般最先被删除。`MaxPerPodContainer`和`MaxContainer`可能会出现冲突，比如在保证每个Pod下容器的数量最大值时（MaxPerPodContainer）却超出了全局能允许的死掉的容器数量（`MaxContainers`）。`MaxPerPodContainer`可以在这种情况下被调整：最坏的情况就是将`MaxPerPodContainer`降到1并剔除最老的容器。而且，当Pod被删除后，当容器年龄超过`MinAge`就会被删除。

那些不由kubelet管理的容器不会被回收。

## 用户设定

用户可以用下面的kubelet选项来调整相关的镜像回收阈值：

- 1.`image-gc-high-threshold`，触发镜像回收的磁盘使用百分比。默认是85%。
- 2.`image-gc-low-threshold`，通过镜像回收希望释放掉的磁盘百分比。默认是80%。

我们还允许用户通过以下kubelet选项来自定义垃圾回收策略：

- 1.`minimum-container-ttl-duration`，已结束的容器在可以被垃圾回收之前的最小年龄。默认是0分钟，也就是或每个结束的容器都会被垃圾回收。
- 2.`maximum-dead-containers-per-container`，每个容器可以保留的旧实例的最大数量。默认是1。
- 3.`maximum-dead-containers`，全局能够保留的容器旧实例的最大数量。默认是-1，也就是没有限制。

容器可以在它们被利用完之前就进行垃圾回收。这些容器会包含日志以及其他数据，便于排错。建议将`maximum-dead-containers-per-container`设置的足够大，这样每个容器至少能保留1个旧实例.同样建议把`maximum-dead-containers`设的更大，原因也是一样。详见[这个issue](https://github.com/kubernetes/kubernetes/issues/13287)。

## 废弃

本文中的一些kubelet垃圾回收特性在未来的版本中可能会被kubelet的剔除功能所替代。

包括：

**现在的选项**|**新的选项**|**原因**
-|-|-
`--image-gc-high-threshold`|`--eviction-hard`或`--eviction-soft`|剔除信号可以触发镜像回收
`--image-gc-low-threshold`|`--eviction-minimum-reclaim`|剔除回收可以实现同样的效果
`--maximum-dead-containers`||当日志存储在容器上下文之外的时候就废弃
`--maximum-dead-containers-per-container`||当日志存储在容器上下文之外的时候就废弃
`--minimum-container-ttl-duration`||当日志存储在容器上下文之外的时候就废弃
`--low-diskspace-threshold-mb`|`--eviction-hard`或`--eviction-soft`|剔除会将磁盘阈值概括到其他资源中
`--outofdisk-transition-frequency`|`--eviction-pressure-transition-period`|剔除会将磁盘压力转移到其他资源

上面这个表格翻译的不太好，你们自求多福。

## 下一步……

更多细节尽在[如何配置资源不足时的应对机制](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)。