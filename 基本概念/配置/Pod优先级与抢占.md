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

## 如何关掉抢占

>**小心**：核心Pod在集群资源紧张的时候需要依赖调度器的抢占来进行调度。所以，不建议关掉抢占。

>**注意**：从1.15版本开始，如果打开了`NonPreemptingPriority`特性，PriorityClass就会设置`preemptionPolicy: Never`。这样这个PriorityClass对应的Pod就不会抢占其他的Pod。

抢占是通过kube-scheduler的`disablePreemption`参数来控制的，默认是`false`。如果你看了上面说的警告还是一门心思想关了它，那你就把它设成`true`。

该选项只能用于组件配置中，并不能用在老式的命令行参数上。下面是个栗子：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider

...

disablePreemption: true
```

## PriorityClass

PriorityClass没有命名空间的概念，它定义了一个优先级类名到整数值的映射关系。其中，名字定义在PriorityClass对象的metadata的`name`字段中。值则是定义在`value`字段。值越大，优先级越高。PriorityClass对象的名字必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)，而且不能以`system-`开头。

PriorityClass对象的值可以是任何小于等于10亿的32位整数。更大的值是保留给关键的系统Pod，一般不允许被抢占或者剔除。集群管理员需要给每一个映射创建一个PriorityClass对象。

PriorityClass还有两个可选字段：`globalDefault`和`description`。`globalDefault`字段就是说这个值会提供给那些没有声明`priorityClassName`的Pod。系统中只能存在一个`globalDefault`字段为true的PriorityClass。如果没有这种PriorityClass，没有定义`priorityClassName`的Pod的优先级就是零。

`description`字段可以是任意的字符串。可以用来告诉用户它的作用。

### 关于已有集群和Pod优先级的几点问题

- 如果你升级后的集群没有开启这个特性，已有Pod的优先级就都相当于零了。
- 如果添加了一个`globalDefault`等于`true`的PriorityClass并不会改变已有Pod的优先级。只有在这个PriorityClass之后创建的Pod才会应用它的优先级。
- 如果你删了一个PriorityClass，使用它的Pod依然没有影响，但是你不能再创建同样的Pod了。

### PriorityClass栗子

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

## 非抢占型PriorityClass

**功能状态**：`Kubernetes v1.15 [alpha]`

如果Pod的`PreemptionPolicy: Never`，那么在调度队列中就会放在更低优先级Pod的前面，但是不会抢占其他Pod。非抢占Pod会一直在调度队列中等待调度，直到拥有足够的资源才可以被调度。非抢占Pod和其他Pod一样要服从调度器的退避算法。这就是说如果调度器尝试调度这些Pod但是发现无法调度，那就会以更低的频率来进行重试，允许其他更低优先级的Pod提前进行调度。

非抢占Pod仍然可以被其他高优先级的Pod抢占。

`PreemptionPolicy`默认值是`PreemptLowerPriority`，允许对应的Pod可以抢占更低优先级的Pod（默认就是这样）。如果`PreemptionPolicy`是`Never`，对应的Pod就是非抢占的。

要想使用`PreemptionPolicy`字段就必须要开启`NonPreemptingPriority`[特性门](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)。

其中一种场景就是那些数据科学的任务。用户提交了一项任务，希望它优先于其他的任务，但又不想让它抢占正在运行的Pod。高优先级的Pod，并且`PreemptionPolicy: Never`，当集群资源“自然而然地”满足需求时，会优先于队列中的其他Pod进行调度。

### 非抢占PriorityClass示例

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

## Pod优先级

当你定义了一个或者多个PriorityClass，你就可以创建Pod并在定义中使用这些PriorityClass。优先级准入控制器会使用`priorityClassName`字段，然后注入对应的优先级整数值。如果没有找到对应的PriorityClass，Pod会被拒绝。

下面YAML中的Pod使用了上文中创建好的PriorityClass。优先级准入控制器会检查这份定义，将Pod的优先级解析成1000000。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

### Pod优先级对调度顺序的影响

如果启用了Pod优先级，调度器会根据Pod的优先级进行排序，高的在前低的在后。结果就是，当需求满足的情况下，高优先级的Pod会比低优先级的Pod更快地落地。如果Pod无法调度，调度器会继续运转并开始调度其他低优先级的Pod。

## 抢占

当Pod创建之后，它们会被放到一个队列中等待调度。调度器从队列中选择一个Pod然后尝试将它调度到一个节点上。如果没有哪个节点能满足这个Pod的需求，它的抢占机制。我们把当前这个Pod称为P。抢占逻辑尝试找到这样一个节点，如果在这个节点上去掉一个或多个比P优先级还要低的Pod就能够让P调度到这个节点。如果找到了这样的节点，就会将一个或者多个比P优先级低的Pod从这个节点上剔除掉。当这些Pod走了之后，P就可以被调度到这个节点上了。

### 用户暴露的信息

当P在节点N上抢占了一个或多个Pod后，P的status中的`nominatedNodeName`字段就会设置成节点N的名字。这个字段可以帮助调度器来跟踪为P保留的资源，还可以给用户提供集群中发生抢占的相关信息。

要注意的是，P并不是一定要被调度到“提名节点”上。当受害者Pod们被抢占了之后，会触发它们的优雅关闭周期。如果当调度器正在等待Pod结束的过程中发现有另一个节点满足了需求，那调度器就会把P调度到另一个节点上。结果就是Pod中的`nominatedNodeName`和`nodeName`并不总是相同的。而且，如果调度器抢占了节点N上的Pod，但此时突然来了一个比P的优先级还高的Pod，那调度器就可能会把这个新的Pod放到节点N上。此时调度器会清除P的`nominatedNodeName`。这样，调度器就可以让P去抢占另一个节点上的Pod了。

### 抢占的限制

#### 受害者们的优雅关闭

当Pod被抢占的时候，受害者会进入它们的[优雅关闭周期](../业务组件/泡德（Pod）/Pod.md#Pod结束状态)。它们在这段时间内可以结束它们的工作然后退出。如果弄不完，它们就会被杀掉。此时的优雅关闭周期代表了从调度器抢占Pod开始到P可以被调度到节点N上的时间。期间调度器依然会调度其他等待中的Pod。当受害者们退出或者被杀掉之后，调度器尝试调度等待队列中的Pod。因此，在调度器开始抢占受害者到P被调度之间，通常都是有一段时间的。为了将这段时间尽量缩短，可以将低优先级Pod的优雅关闭时间调小一点或者直接改成零。

#### 支持PodDisruptionBudget但它并不受保障

[PodDisruptionBudget](../业务组件/泡德（Pod）/分裂.md)（PDB）可以限制多副本应用在主动分裂的时候同时停掉的副本数量。当抢占发生的时候也支持PDB，但也只是尽最大努力去满足PDB的要求。调度器在抢占的时候会尝试找到不会违反PDB要求的受害者，但是如果没有这样的受害者，抢占依然会照常进行，不论是否违反了PDB的要求，低优先级的Pod依然会被剔除。

#### 低优先级的Pod间亲和性

只有下面这个问题的回答是yes的情况下节点才会被认为是可以抢占的：“如果节点上所有比当前Pod优先级低的Pod都被剔除了，此时P是否能调度到这个节点上？”

>**注意**：抢占并不需要剔除所有较低优先级的Pod。如果当前Pod想要被调度只需要剔除某几个Pod就可以，那就只会剔除这几个Pod。即便如此，上面问题的回答依然得是yes。如果答案是no，该节点就不会发生抢占。

如果当前Pod跟节点上的一个或多个低优先级Pod之间存在亲和性，少了这些Pod就无法满足亲和性的规则。这种情况下，调度器不会抢占该节点的任何Pod。它会寻找另一个节点。调度器可能能，也可能不能找到下一个合适的节点。无法保证当前Pod一定能够成功调度。

对于这种问题，我们建议的解决方法是只对同优先级或者更高优先级的Pod建立亲和性。

#### 跨节点抢占

假设节点N要发生抢占了，然后P就可以调度到节点N上了。此时会发生的一种情况是，可能只有当另一个节点上的一个Pod被抢占后，P才能够调度到节点N上。让我们来举个栗子：

- 准备将P落到节点N上。
- Q运行在另一个节点上，这个节点跟节点N处于同一个可用区中。
- P和Q之间存在基于可用区的反亲和性（`topologyKey: failure-domain.beta.kubernetes.io/zone`）。
- 在该可用区中，不存在其他Pod跟P有反亲和性。
- 为了把P调度到节点N上，可以抢占Q，当时调度器不会进行跨节点的抢占。所以P被认为是无法调度到N上了。

如果Q从它的节点上删掉了，反亲和性就可以满足了，P就可能被调度到节点N上了。

如果这种需求越来越多的话，我们在之后的版本中就会考虑增加跨节点的抢占，而且我们还得找到一种满足性能要求的算法。

## 排错

Pod优先级和抢占可能会导致意想不到的副作用。下面是一些可能发生的问题以及处理方法。

### 发生了不需要的抢占

集群资源紧张的情况下抢占机制会删除低优先级的Pod，给高优先级的Pod腾出空间。如果你把高优先级赋给了错误的Pod，这些捣乱的Pod就可能导致集群中发生抢占。Pod优先级是通过在定义中设置`priorityClassName`字段来实现的。优先级对应的整数值会被解析出来然后注入到`podSpec`的`priority`字段中。

为了凸显这个问题，你可以把这些Pod的`priorityClassName`设置成较低优先级的PriorityClass，或者直接把它留空。默认是把空的`priorityClassName`解析成零。

当Pod被抢占的时候，被抢占的Pod会生成事件记录。只有当集群资源无法满足当前Pod的时候才会发生抢占。此时，只有当前Pod（发起抢占者）比受害者Pod的优先级高，才会发生抢占。如果没有等待中的Pod，或者当前Pod优先级和受害者的优先级相等或者更低，那就不会发生抢占。如果此时出现了抢占，那你一定要给我们提一个issue。

### Pod被抢占了，但是抢占者没有被调度

当Pod被抢占时，它们会进入它们设定的优雅关闭周期，默认是30秒。如果受害者们在这段时间内没有结束，那就会被强杀。当所有受害者都离开了，抢占者就可以被调度了。

当抢占者等待受害者离去的时候，可能突然创建了一个更高优先级的Pod，同样也适合当前正在抢占的这个节点。此时，调度器会调度这个更高优先级的Pod，而不是之前的抢占者。

这种行为是说得通的：高优先级Pod就是应该在低优先级的前面。其他的控制器动作，比如[集群自动扩容](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/cluster-management/#cluster-autoscaling)，可能最终可以为当前Pod提供出足够的空间。

### 高优先级Pod比低优先级Pod提前被抢占

调度器会找到合适的节点来调度当前Pod。如果没找到，为了给当前Pod腾出空间，调度器会从任意的一个节点上尝试删除低优先级的Pod。如果低优先级Pod所在的节点不适合当前的Pod，调度器会选择其他拥有更高优先级Pod的节点（和其他节点上的Pod相比）。当然，最终的受害者们的优先级肯定是要比抢占者更低的。

如果适合抢占的节点有多个的话，调度器会选择Pod优先级最低的一组所在的节点。但是，如果这些Pod有PodDisruptionBudget会被违反，那么调度器就要继续选择其他更高优先级Pod所在的节点。

当有多个节点可以进行抢占，而且没有发生以上情况，调度器就会选择优先级最低的那个节点。

## Pod优先级和QoS

Pod优先级和[QoS class](https://v1-18.docs.kubernetes.io/docs/reference/glossary/?all=true#term-qos-class)是两个正交的功能，不存在交互，而且也不会根据QoS class给Pod优先级加以限制。当调取其选择抢占目标的时候不会考虑到QoS。抢占只会看Pod的优先级，选择优先级最低的目标。只有当删除低优先级Pod无法满足抢占者的时候才会考虑抢占更高优先级的Pod，或者说低优先级的Pod受到`PodDisruptionBudget`的保护。

唯一会同时考虑QoS和Pod优先级的情况是[kubelet在资源不足时触发剔除](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)。kubelet在剔除Pod的时候首先要根据它们对当前紧张的资源的使用量是否超过了它们初始的请求值来进行排序，然后时根据优先级排序，然后是Pod请求的计算资源使用情况。详见[剔除用户的Pod](https://v1-18.docs.kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#evicting-end-user-pods)。

如果Pod资源的使用量没有超过它的请求值，在资源不足的时候，kubelet也不会剔除这些Pod。如果这种Pod的优先级较低，也依然不会被剔除。优先级较高，同时资源使用又超过了请求值，那么就可能会被剔除。

## 下一步……

看看和PriorityClass相关的ResourceQuota：[限制默认可用的PriorityClass](../策略/资源配额.md#限制默认的PriorityClass)