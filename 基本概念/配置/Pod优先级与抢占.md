# Pod优先级与抢占

**功能状态**：`Kubernetes v1.14 [stable]`

[Pod](../业务组件/泡德（Pod）/Pod.md)是有*优先级*的。优先级就意味着一个Pod相对于其他Pod的重要性。如果一个Pod无法调度，调度器会试着抢占（剔除）低优先级的Pod，给其他Pod让出一线生机。

>**警告**：
>
>如果集群中并不是所有用户都值得信赖，一个充满邪恶的用户可能会创建拥有最高优先级的Pod，导致其他Pod无法调度或者被剔除。管理员可以使用资源配额（ResourceQuota）阻止用户创建这么高优先级的Pod。
>
>详见[限制默认的PriorityClass](../策略/资源配额.md#限制默认的PriorityClass)。


## 如何使用优先级和抢占

要想使用优先级和抢占：

- 1.至少要有一个[PriorityClass](#PriorityClass)。
- 2.给Pod设置[`priorityClassName`](#Pod优先级)，指向某个PriorityClass。当然了，你用不着直接创建Pod，一般都是用Deployment这种东西，在Pod模板中加上`priorityClassName`。

仔细了解这两步操作中的相关知识。

>**注意**：k8s中已经预置了两个PriorityClass：`system-cluster-critical`和`system-node-critical`。这俩都是常用的类型，用来[保证关键的组件一定要优先调度](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)。

## 

## PriorityClass

## Pod优先级