# Pod生命周期

本文为您介绍Pod的生命周期，看我，多有礼貌，为您！啧啧~

- [各个阶段](#各个阶段)
- [Condition](#Condition)
- [容器探针](#容器探针)
- [Pod和容器的Status](#Pod和容器的Status)
- [容器状态](#容器状态)
- [Pod就绪](#Pod就绪)
- [重启策略](#重启策略)
- [我能活多久](#我能活多久)
- [栗子](#栗子)
- [接下来……](#接下来)

## 各个阶段

Pod的`status`属性是一个[PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podstatus-v1-core)对象，包含一个`phase`字段。

Pod所处的每一个阶段（phase），是对它整个生命周期的一个简单的、宏观的描述。每个阶段并不是对容器或Pod状态的汇总，不能当作综合状态机。（有点学术了？）

Pod所处的阶段数量有严格的限制。除了本文提到的，绝对不要假设Pod的`phase`还有其他值。如果有人跟你说还有XXX，你就当他是个骗子。

下面是`phase`的所有可能值：

值|描述
-|-
`Pending`|k8s系统已经收到这个Pod了，但是容器的镜像还没创建好。这段时间包含了调度所需的时间，以及从网络拉取镜像的时间，可能得耗一阵子。
`Running`|Pod跟节点的关系已经建立起来了，容器也都创建好了。至少有一个容器处于运行或启动中、重启中的状态。
`Succeeded`|Pod中所有容器都成功地完成了任务并结束了，并且不会被重启。
`Failed`|Pod中所有容器都结束了，至少有一个容器的结束状态有问题。也就是说，容器的退出码不等于0，或者是被系统操作系统杀掉了。
`Unknown`|出于某种原因，无法获取到Pod的状态，一般是由于无法跟Pod所在的主机建立通信导致的。

## Condition

每个Pod有一个PodStatus，里面是一个[PodCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podcondition-v1-core)数组，Pod可能通过，也可能没有通过这些Condition。PodCondition数组中的每个元素包含六个可能的字段：

- `lastProbeTime`，该condition最后一次探测的时间戳。
- `lastTransitionTime`，记录Pod最后一次status改变的时间戳。
- `message`，人类可读的该condition的详细信息。
- `reason`，一个唯一的驼峰命名单词，记录上一次condition发生变化的原因。
- `status`，一个字符串，可能的值包括“`True`”、“`False`”、“`Unknown`”。
- `type`，一个字符串，可能值包括：
    - `PodScheduled`：Pod已经被调度到一个节点上了；
    - `Ready`：Pod已经可以开始接流量了，所有匹配的Service都可以将其加入到负载均衡池中了；
    - `Initialized`：所有[初始化容器]()已经正常启动；
    - `ContainersReady`：Pod中所有容器都准备好了。

## 容器探针

[容器探针（Probe）](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#probe-v1-core)是[kubelet]()对容器进行的周期性状态诊断。执行诊断时，kubelet需要调用容器提供的[Handler](https://godoc.org/k8s.io/kubernetes/pkg/api/v1#Handler)。一共三种Handler：

- [ExecAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#execaction-v1-core)：执行一个容器中的命令。如果命令退出码为0，则诊断正常。
- [TCPSocketAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#tcpsocketaction-v1-core)：容器的IP加一个指定的端口，对其进行TCP检查。端口能打开，则诊断正常。
- [HTTPGetAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#httpgetaction-v1-core)：容器的IP加一个指定的端口和路径，对其发起HTTP的Get请求。如果HTTP返回码大于等于200且小于400，则诊断正常。

每次探测有三种可能的结果：

- Success：探测结果正常。
- Failure：探测结果失败。
- Unknown：探测本身出现问题，不执行任何动作。

上面说的是具体执行探测时的动作，下面讲一下探测本身的种类，也有三种，可以让kubelet根据这三种探测的结果执行一些响应动作：

- `livenessProbe`：代表容器是否在运行。如果该探测结果失败，则kubelet会将容器杀死，然后容器参考自身的[重启策略](#重启策略)执行相应的动作。如果容器没有提供这种探测，默认状态为`Success`。
- `readinessProbe`：代表容器是否可以开始接流量了。如果该探测结果失败，端点控制器会将该Pod的IP从所有匹配的服务（Service）中删掉。在初始化完成之前，该探测的默认值是`Failure`。如果容器没有提供这种探测，默认状态为`Success`。
- `startupProbe`：代表容器内的应用程序是否已经启动完成了。如果容器提供了这种探测，在该探测结果正常之前，其他两种探测都会被关掉。如果该探测失败，则kubelet会将容器杀死，然后容器参考自身的[重启策略](#重启策略)执行相应的动作。如果容器没有提供这种探测，默认状态为`Success`。

### 啥时候需要存活探测（livenessProbe）

**功能状态**：`Kubernetes v1.0`（stable）

如果容器中的进程因为某些原因会导致肚子疼或者直接崩溃，那你其实不用加存活探测，因为kubelet就会自动根据Pod的`restartPolicy` 来做出正确的响应。

如果在某种探测结果失败的情况下你希望容器被杀掉然后重启，那你就把存活探测加上，然后记得把`restartPolicy`设置为Always或OnFailure。

### 啥时候需要就绪探测（readinessProbe）

**功能状态**：`Kubernetes v1.0`（stable）

如果在某种探测结果正常的情况下才可以把流量交给Pod，那就加上就绪探测。此时，就绪探测跟存活探测好像有点撞衫了，但存在即合理，就绪探测的作用就是说，容器起来是起来了，但是只有在某个探测结果正常的情况下，才能开始接流量。如果容器在起动过程中需要加载大量数据、配置文件，或者要做什么迁移，那你就把就绪探测加上。

如果在进行维护的时候，你希望容器能自己从车里出来，那就可以为就绪探测指定一个有别于存活探测的端点。

注意了，如果你只是说想在Pod被删除的时候把流量卸掉，那跟就绪探测就没什么关系了；删Pod的时候，Pod自动就转入非就绪状态了，不管有没有就绪探测。Pod会保持在非就绪状态，等着容器停掉。

### 啥时候需要启动探测（startupProbe）

**功能状态**：`Kubernetes v1.16`（alpha）

如果容器启动所需时间总是大于`initialDelaySeconds + failureThreshold × periodSeconds`，那就加一个启动探测，跟存活探测指向同一个端点。`periodSeconds`的默认值是30秒。把`failureThreshold`调到足够大，让容器正常启动，这样就不用去改存活探测的默认值了。这种用法可以处理一些死锁的情况。

关于设置各种探测，详见[这里]()。

## Pod和容器的Status

详见[PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podstatus-v1-core)跟[ContainerStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#containerstatus-v1-core)。注意了，Pod的status依赖于当前的[ContainerState]()。

## 容器状态

Pod调度到节点后，kubelet就开始用容器运行时来创建容器了。容器会有三种状态：Waiting、Running和Terminated。可以用`kubectl describe pod [Pod名]`来查看容器的状态。Pod中每个容器的状态会分别显示。

- `Waiting`：容器的默认状态。如果容器不是Running或者Terminated，那它就是Waiting。Waiting时的容器依然在进行着必要的工作，比如拉镜像、申请Secret等。这种状态下，会显示一条状态相关的信息。

```text
...
  State:          Waiting
   Reason:       ErrImagePull
...
```

- `Running`：容器跑起来了，没啥问题。在容器转入Running状态前，会先执行`postStart`钩子（如果有的话）。此时还会显示转入Running状态的时间。

```text
...
  State:          Running
   Started:      Wed, 30 Jan 2019 16:46:38 +0530
...
```

- `Terminated`：容器的工作都执行完了，停止运行了。此时，要么圆满完成了它的工作，要么就是因为啥原因出事儿了。不论怎么着吧，此时都会显示对应的原因和退出码，以及容器的启动和结束时间。在容器转入Terminated状态前，会先执行`preStop`钩子（如果有的话）。

```text
...
  State:          Terminated
    Reason:       Completed
    Exit Code:    0
    Started:      Wed, 30 Jan 2019 11:45:26 +0530
    Finished:     Wed, 30 Jan 2019 11:45:26 +0530
...
```

## Pod就绪

**功能状态**：`Kubernetes v1.14`（alpha）

应用程序可以将额外的信息注入到PodStatus中：即*Pod readiness*。上面我们讲的是容器的readiness，现在是Pod的readiness。使用时，在PodSpec中设置`readinessGates`，添加额外的condition。

就绪（readiness）状态是由Pod的`status.condition`字段中的当前状态来决定的。如果在`status.conditions`字段中没有找到对应的condition，该condition则默认为“`False`”。示例如下：

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

condition的名字必须符合k8s的[标签key格式](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#规范)。

### Pod就绪状态

`kubectl patch`命令无法对对象状态进行修补（patching）。要给Pod设置`status.conditions`，应用程序或者[操作器（operator）]()应该使用`PATCH`动作。可以用[k8s客户端库]()，通过编码的方式给Pod设置自定义的condition。

对于使用了自定义condition的Pod，Pod进入就绪状态**必须**达成以下两个条件：

- Pod中所有的容器都就绪了。
- 所有`ReadinessGates`中声明的condition都是`True`。

## 重启策略

PodSpec中有一个`restartPolicy`字段，可能的值包括Always、OnFailure和Never。默认是Always。`restartPolicy`适用于Pod中的所有容器。kubelet使用`restartPolicy`来重启本节点的容器。重启基于指数退避算法（10秒，20秒，40秒……），5分钟间隙，成功执行十分钟后重置。（一到这个算法我就翻译不明白）。正如[Pod文档](Pod.md#Pod持久化（或者说没有持久化）)所述，一旦Pod跟一个节点绑定起来，就不会再绑定到其他节点了，挺专一的。

## 我能活多久

除非人类，或者[控制器](../../集群架构/控制器.md)把它删了，否则一般情况下Pod都会一直活下去。当尸体数量超过阈值（kube-controller-manager的`terminated-pod-gc-threshold`），control plane就要清理这些尸体Pod（阶段（phase）值为`Succeeded`或`Failed`）。随着Pod一波一波来，一波一波走，这样可以避免资源泄漏。

有多种可以创建Pod的资源：

- 对于不停歇的那种Pod，比如Web服务，可以用[Deployment]()、[ReplicaSet]()、[StatefulSet]()。
- 对于任务完成就可以滚蛋的Pod，可以用[Job]()；比如那种跑批的业务。Job只适用于`restartPolicy`值为OnFailure或Never的Pod。
- 每个节点都想运行一个Pod，可以用[DaemonSet]()。

上面列出的所有资源，都包含一个PodSpec。建议使用这些资源来管理Pod，而不是你自己直接去搞。

如果节点去世了，或者失联了，k8s会将该节点上的所有Pod的`phase`设置为Failed。

## 栗子

### 超牛逼的存活探测

存活探测（liveness probe）由kubelet来执行，因此，所有请求都在kubelet的网络空间中进行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: k8s.gcr.io/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      timeoutSeconds: 1
    name: liveness
```

### 各种状态举例

- Pod运行中，只有一个容器。容器正常退出。
    - 记录完成事件。
    - 如果`restartPolicy`是：
        - Always：重启容器；`phase`保持在Running。
        - OnFailure：`phase`转入Succeeded。
        - Never：`phase`转入Succeeded。
- Pod运行中，只有一个容器。容器异常退出。
    - 记录异常事件。
    - 如果`restartPolicy`是：
        - Always：重启容器；`phase`保持在Running。
        - OnFailure：重启容器；`phase`保持在Running。
        - Never：`phase`转入Failed。
- Pod运行中，有两个容器。容器1异常退出。
    - 记录异常事件。
    - 如果`restartPolicy`是：
        - Always：重启容器；`phase`保持在Running。
        - OnFailure：重启容器；`phase`保持在Running。
        - Never：`phase`保持在Running。
    - 如果容器1不处于运行中，容器2退出：
        - 记录异常事件。
        - 如果`restartPolicy`是：
            - Always：重启容器；`phase`保持在Running。
            - OnFailure：重启容器；`phase`保持在Running。
            - Never：`phase`转入Failed。
- Pod运行中，只有一个容器。容器内存不足。
    - 容器异常退出。
    - 记录OOM事件。
    - 如果`restartPolicy`是：
        - Always：重启容器；`phase`保持在Running。
        - OnFailure：重启容器；`phase`保持在Running。
        - Never：记录异常事件；`phase`转入Failed。
- Pod运行中，有块儿磁盘坏了。
    - 杀死所有容器。
    - 记录相应事件。
    - `phase`转入Failed。
    - 如果有负责的控制器，Pod会在其他地方重建。
- Pod运行中，节点被切掉了。
    - 节点控制器进入超时等待。
    - 节点控制器将Pod的`phase`设为Failed。
    - 如果有负责的控制器，Pod会在其他地方重建。

## 接下来……

- 实操：为容器生命周期事件设置handler。
- 实操：设置存活、就绪、启动探测。
- 学习[容器的生命周期钩子](../../容器/容器生命周期钩子.md)。