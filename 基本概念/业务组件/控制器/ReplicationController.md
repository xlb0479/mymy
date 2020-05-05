# ReplicationController

>**注意**：需要副本功能时，[`Deployment`]()是更好的选择，它自带[`ReplicaSet`](ReplicaSet.md)。这里的内容你学学就行了，别当个宝似的。

*ReplicationController*可以保证在任意时刻有指定数量的Pod副本在运行。换句话说，ReplicationController可以保证一个或者一组同构的Pod一直处于可用状态。

- [咋工作的](#咋工作的)
- [跑一个例子](#跑一个例子)
- [Spec规范](#Spec规范)
- [各种操作](#各种操作)
- [常见用法](#常见用法)
- [开发支持副本的程序](#开发支持副本的程序)
- [它的职责](#它的职责)
- [API对象](#API对象)
- [其他方案](#其他方案)
- [更多](#更多)

## 咋工作的

如果Pod过多，ReplicationController会停掉多余的。如果Pod过少，ReplicationController会创建新的。不同于手动创建Pod，ReplicationController下的Pod在出现问题、被删除、被停掉之后会自动创建新的Pod来替换。比如节点内核需要升级维护，节点分离后，上面的Pod会自动在其他节点上重建。所以即便你只需要一个Pod也应该使用ReplicationController。ReplicationController的作用有点类似于进程的Supervisor，只不过是负责管理多个节点上的多个Pod，而不是某个主机上的某个进程。

ReplicationController经常简写成“rc”，在kubectl命令中也可以使用这种简写。

简单的用法就是建一个ReplicationController，可靠地让一个Pod持续运行。稍微复杂点的用法就是运行一个服务的多个相同的副本，比如Web服务器。

## 跑一个例子

下面的例子跑了三个Nginx副本。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

可以直接运行网络上的yaml：

```shell script
kubectl apply -f https://k8s.io/examples/controllers/replication.yaml
```

```text
replicationcontroller/nginx created
```

查看ReplicationController的状态：

```shell script
kubectl describe replicationcontrollers/nginx
```

```text
Name:        nginx
Namespace:   default
Selector:    app=nginx
Labels:      app=nginx
Annotations:    <none>
Replicas:    3 current / 3 desired
Pods Status: 0 Running / 3 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=nginx
  Containers:
   nginx:
    Image:              nginx
    Port:               80/TCP
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen       LastSeen     Count    From                        SubobjectPath    Type      Reason              Message
  ---------       --------     -----    ----                        -------------    ----      ------              -------
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-qrm3m
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-3ntk0
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-4ok8v
```

可以看到，建了三个Pod，但都还没跑起来，可能是还在拉镜像吧。我们再等一会儿，执行同样的命令可以看到：

```text
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

如果想以机器可读的格式列出ReplicationController包含的Pod，可以这样：

```shell script
pods=$(kubectl get pods --selector=app=nginx --output=jsonpath={.items..metadata.name})
echo $pods
```

这里的选择器和ReplicationController里的一样（看`kubectl describe`命令的输出结果），但是跟`replication.yaml`文件中的格式不太一样。`--output=jsonpath`通过表达式参数要求返回结果只包含每个Pod的名字。

## Spec规范

跟k8s的其他配置一样，ReplicationController也要有`apiVersion`、`kind`、`metadata`字段。ReplicationController对象的名字必须是有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。关于这些配置的一般信息，可以看看[对象管理](../../概要/Kubernetes对象/k8s对象管理.md)。

ReplicationController同样还需要一个[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)。

### Pod模板

在`.spec`中需要的唯一字段就是`.spec.template`。

`.spec.template`是一个[Pod模板](../泡德（Pod）/概要.md#Pod模板)。其格式跟[Pod](../泡德（Pod）/Pod.md)一样，除了它是嵌套在别的字段里的，而且没有`apiVersion`和`kind`。

除了Pod必须的字段外，ReplicationController的Pod模板还需要指定合适的标签跟重启策略。标签要注意不要跟其他控制器冲突。见[Pod选择器](#Pod选择器)。

`.spec.template.spec.restartPolicy`的值只能是`Always`，这也是默认值。

本地容器的重启，ReplicationController会将它委托给节点上的代理，比如[Kubelet]()或Docker。

### ReplicationController的标签

ReplicationController本身也可以有标签（`.metadata.labels`）。一般来说这个标签要设置成跟`.spec.template.metadata.labels`一样的值；如果没写`.metadata.labels`，默认就是跟`.spec.template.metadata.labels`一样。但是它们是可以不同的，`.metadata.labels`的值不会影响到ReplicationController的行为。

### Pod选择器

`.spec.selector`是一个[标签选择器](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)。ReplicationController管理着跟这个选择器匹配的所有Pod。它并不区分Pod是它自己来建/删还是其他人/进程来建/删。这样就可以在不影响Pod的情况下对ReplicationController进行替换。

如果定义了`.spec.template.metadata.labels`就必须匹配`.spec.selector`，否则API报错。如果没有`.spec.selector`，默认就跟`.spec.template.metadata.labels`一样。

还有一个要注意的地方就是不要建一些跟ReplicationController选择器匹配的Pod，不管是手动建还是用其他ReplicationController、Job来建。如果你这么干，k8s不会拦着你，但是其他的匹配的ReplicationController会把这些Pod当成自己下的蛋。

如果你真的陷入了多个控制器的选择器重叠的情况，那你就得自己去删除（[见下文](#删除ReplicationController和它的Pod)）。

### 多个副本

可以用`.spec.replicas`来指定并发运行的Pod数量。具体到某个时间点的话，数量可能在这个值的上下浮动，比如副本数增加或减少，或者Pod正在优雅关闭，而另一个替代的Pod已经起来了。

`.spec.replicas`默认值为1。

## 各种操作

### 删除ReplicationController和它的Pod

可以用[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)来删除ReplicationController和它的Pod。Kubeclt会先将ReplicationController的规模降到零，等它把所有Pod都删了，然后再删掉它本身。如果执行命令过程被中断，可以重复执行。

如果是用REST API或者Go客户端库，这些操作你得自己手动进行（副本降0，等Pod被删，删除ReplicationController）。

### 只删除ReplicationController

可以只删除ReplicationController而不影响它的Pod。

使用[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)命令，增加`--cascade=false`参数。

如果是用REST API或者Go客户端库，那就直接删ReplicationController就行了。

删了之后，你可以建一个新的来替代它。只要`.spec.selector`不变，就还是那些Pod。但是如果Pod模板变了，原来的Pod不会更新。如果要以可控的方式更新Pod定义，请享用[滚动更新](#滚动更新)。

### 从ReplicationController中分离Pod

可以通过修改Pod的标签，将Pod从ReplicationController中剔除。这种高新科技可以用在需要调试或数据恢复等场景中，分离出一个Pod。剔除之后会立即有新的Pod被创建（假设我们没有修改副本数量需求）。

## 常见用法

### 重新调度

如前文所讲，不管你是要1个还是1000个Pod保持运行，都可以用ReplicationController来确保数量正确，即便是节点出错或Pod停止（可能由于其他控制程序导致）。

### 伸缩

ReplicationController可以轻而易举对副本数进行伸缩，不管是手动还是通过自动伸缩控制程序，只需要修改`replicas`字段就可以了。

### 滚动更新

ReplicationController可以辅助服务进行滚动更新，一个一个地对Pod进行替换。

正如[#1353](https://github.com/kubernetes/kubernetes/issues/1353)中的讨论，关于滚动更新，推荐的方法是，先创建一个只有1个副本的新的ReplicationController，然后每次新的副本数+1，旧的-1，一个一个替换，最后旧的ReplicationController副本数降到0，就可以删除了。这种方式可以避免一些意料之外的问题。

理想条件下，负责滚动更新的控制器应该考虑到Pod的就绪情况，确保任何时刻都有足够的Pod数量来处理业务负载。

一新一旧两个ReplicationController至少要有一个不同的标签，比如主要容器的镜像Tag，因为一般都是因为镜像要更新才去闹滚动更新。

### 多版本轨迹

除了在滚动更新过程中可能要同时运行应用的多个版本外，多版本应用通常还要多运行一段时间，甚至是保持多个版本轨迹持续地运行下去。这些轨迹可以从标签上区分出来。

比如一个服务对应的Pod是`tier in (frontend), environment in (prod)`。假设现在这个条件下有10个Pod副本。此时这个服务要做一个灰度发布（Canary）。可以创建一个`replicas`值为9的ReplicationController，标签选择`tier=frontend, environment=prod, track=stable`，另一个`replicas`值为1，标签选择`tier=frontend, environment=prod, track=canary`。此时这个服务下就既有灰度版本也有正式版本。你可以用ReplicationController来进行测试，监控灰度结果等等。

### 组合ReplicationController与服务（Service）

一个服务后面可以有多个ReplicationController，因此，这种情况下，一部分流量走老版本，一部分走新版本。

一个ReplicationController从来不会自己停掉，但也不是像Service那么长寿。Service可能由多个由不同ReplicationController控制的Pod组成，而这些ReplicationController在Service的一生中可能也是死了一个又来一个（比如要更新Service的Pod）。ReplicationController对于客户端和Service来说都应该是透明的。

## 开发支持副本的程序

尽管配置上可能随着时间流逝会发生一些变化，但是由ReplicationController创建的Pod本质上应当是可替代的，语义不发生变化的。这一点在对无状态服务进行复制时是比较明显的优点，但是ReplicationController还可以用来维护那些需要进行主节点选举的，分片的，有工作节点池的程序的可用性。这种程序都会有动态工作量分配机制，比如[RabbitMQ 工作队列](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)，这与那种静态的/一次性的Pod配置正相反，被认为是反模式。任何Pod的自定义，比如资源的垂直伸缩（CPU或内存等），应该由另一个控制器程序来执行，这跟ReplicationController基本上是一样的。

## 它的职责

ReplicationController只是负责保障它负责的Pod保持在正确的副本数。目前来说只有那些停掉的Pod不会被计算在内。后期可能会将[就绪状态](https://github.com/kubernetes/kubernetes/issues/620)以及其他信息考虑在内，可能会对替换策略增加更多的控制，而且还准备触发一些事件，这样一些外部的客户端可以根据这些事件来实现任意复杂的替换/缩容策略。

ReplicationController这辈子的职责也就是这么多了。它本身不去做就绪或存活探测。还有自动伸缩，也是应该由外部的自动伸缩程序来控制（见[#492](https://github.com/kubernetes/kubernetes/issues/492)），由它们来修改`replicas`字段。我们不会给ReplicationController增加调度策略（比如[分布限制](https://github.com/kubernetes/kubernetes/issues/367#issuecomment-48428019)）。同样，不会让它去验证当前控制的Pod是否跟当前的模板匹配，这样会影响到自动伸缩以及一些其他的自动化程序。还有，比如什么完成期限、顺序依赖、配置扩展，巴拉巴拉小魔仙等等都是其他组件的功能。我们甚至还想将Pod的批量创建工作分出去（[#170](https://github.com/kubernetes/kubernetes/issues/170)）。

ReplicationController目的是作为一种组合式的分块构建原语。我们希望能有更高层的API或工具基于它进行构建，以及未来通过其他的互补的原语为用户提供便利。kubectl目前支持的“marco”操作就是这种例子（run、scale）。具体比如我们可以想象通过[Asgard](https://netflixtechblog.com/asgard-web-based-cloud-management-and-deployment-2c9fc4e4d3a1)这种工具来管理ReplicationController、自动伸缩程序、服务、调度策略、灰度发布等。

## API对象

ReplicationController是k8s REST API中的顶层资源。关于API对象的更多信息可以去看：[ReplicationController API对象](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#replicationcontroller-v1-core)。

## 其他方案

### ReplicaSet
[`ReplicaSet`](ReplicaSet.md)是下一代ReplicationController，支持[基于条件集合的标签选择器](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#包含性选择)。这个东西主要是帮[`Deployment`](Deployment.md)对Pod的撞见、删除和更新进行一些流程编排。你注意一下，我们建议用Deployment代替ReplicaSet，除非你准备自己闹更新相关的策略，或者你压根不需要更新。

### Deployment（推荐）

[`Deployment`](Deployment.md)是更高层的API对象，它可以更新底层的ReplicaSet以及Pod。如果你想用滚动更新，那就推荐你用Deployment，因为它本身是声明式的，基于服务端运行的，并且还有其他小惊喜。

### 果Pod

不同于用户直接创建Pod，ReplicationController可以在Pod被删除或因错误、节点维护等情况下Pod被停掉时，自动替换Pod。正因为如此，即便你只需要一个Pod，我们也建议你用ReplicationController。可以把它当成进程的Supervisor，只不过它管理的时多个跨节点的Pod，而不是一个节点上某个进程。ReplicationController将本地容器的重启工作委托给节点上的代理（比如Kubelet或Docker）。

### Job

对于那种需要自然终止的Pod，可以用[`Job`](Job.md)来代替ReplicationController（比如批处理任务）。

### DaemonSet

对于想要提供机器级别功能的Pod，比如主机监控和日志记录，可以用[`DaemonSet`]()代替ReplicationController。这种Pod的存亡是跟底层主机联系起来的：应当在其他Pod运行之前就先运行这种Pod，并且在主机准备重启/关机的时候能安全的停止。

## 更多

[用ReplicationController运行无状态应用]()