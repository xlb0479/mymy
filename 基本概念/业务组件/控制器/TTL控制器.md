# 资源结束后的TTL控制器

**功能状态**：`Kubernetes v1.12 [alpha]`

对于那些已经执行完成的资源对象，TTL控制器可以提供一个TTL（time to live）存活时间机制，来限制资源还能活多久。目前，TTL控制器只会处理[Job](Job.md)，以后可能会继续扩展到其他执行完成的资源上，比如Pod或者自定义的资源。

Alpha声明：这个功能目前还是阿尔法呢！阿尔法狗呢！可以通过kube-apiserver、kube-controller-manager的`TTLAfterFinished`[特性门]()开启。

- [TTL控制器](#TTL控制器)
- [注意事项](#注意事项)
- [接下来……](#接下来)

## TTL控制器

目前TTL控制器只支持Job。集群操作者们可以用这个功能，给Job设置`.spec.ttlSecondsAfterFinished`字段，在Job结束（`Complete`或`Failed`）后自动进行清理工作，见[这个栗子]()。资源运行结束TTL秒后，也就是到达TTL超时时间后，TTL控制器会假设资源都是可以进行清理的，一副自以为是的嘴脸。TTL在进行清理的时候会对资源进行级联删除，会删除对象和对象的依赖。注意，当资源被删除时，它的生命周期中的各个保障机制，比如finalizer，都会按照正规流程去走。

可以随时设置TTL时间。下面给出一些`.spec.ttlSecondsAfterFinished`的栗子：

- 在资源定义时就声明这个字段，这样Job结束并到达TTL后就会自动清理。
- 为已经结束的资源设置这个字段，依然能够生效。
- 使用[mutating admission webhook]()，在资源创建时动态设置TTL字段。集群管理员可以通过这种方式为已结束的资源强制执行TTL策略。
- 使用[mutating admission webhook]()，在资源结束时动态设置TTL字段，可以根据资源的状态、标签等选择不同的TTL值。

## 注意事项

### 更新TTL时间

注意，`.spec.ttlSecondsAfterFinished`的值可以在资源创建、结束后进行修改。但是当Job被认为是可删除时（到达TTL），系统不会保证Job依然健在，即便你请求延长Job的TTL，并且API也给你返回OK了，也依然不能保证真的生效。

### 时间倾斜

TTL控制器是基于存储在k8s资源中的时间戳来判断资源是否到达TTL，这就很容易受到集群时间倾斜的影响，可能会导致TTL控制器在错误的事件进行清理。

在k8s中需要在所有节点上用NTP来避免时间倾斜（见[#6159](https://github.com/kubernetes/kubernetes/issues/6159#issuecomment-93844058)）。时钟可能并不总是正确，但差距应该非常小。当你设置了TTL后，一定要注意这个问题。

## 接下来……

[自动清理Job]()

[设计文档](https://github.com/kubernetes/enhancements/blob/master/keps/sig-apps/0026-ttl-after-finish.md)