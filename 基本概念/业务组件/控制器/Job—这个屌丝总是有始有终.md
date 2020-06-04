# Job—这个屌丝总是有始有终

一个Job会创建若干个Pod，并且能保证其中某几个最终一定会成功运行结束。Pod成功地完成后，Job会跟踪到这个状态。当成功完成的数量达到指定阈值时，这个任务（Job）也就完成了。删除Job也会删除它创建的那些Pod。

一个简单的用法是，我们创建一个Job对象，是为了让一个Pod能够可靠地、成功地运行完成。如果第一个Pod失败或者被删了（比如节点硬件异常或者重启），Job会自动创建一个新的Pod。

还可以用Job并行地运行多个Pod。

- [跑个栗子](#跑个栗子)
- [编写Job定义](#编写Job定义)
- [Pod和容器的错误处理](#Pod和容器的错误处理)
- [Job的停止与清理](#Job的停止与清理)
- [自动清理已完成的Job](#自动清理已完成的Job)
- [Job设计模式](#Job设计模式)
- [高级用法](#高级用法)
- [其他方案](#其他方案)
- [Cron](#Cron)

## 跑个栗子

本例中，我们计算π小数点后的2000位并打印。大概运行了10秒。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

运行栗子：

```shell script
kubectl apply -f https://kubernetes.io/examples/controllers/job.yaml
```

```text
job.batch/pi created
```

用`kubectl`检查Job的状态：

```shell script
kubectl describe jobs/pi
```

```text
Name:           pi
Namespace:      default
Selector:       controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
Labels:         controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
                job-name=pi
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"default"},"spec":{"backoffLimit":4,"template":...
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Dec 2019 15:20:11 +0200
Completed At:   Mon, 02 Dec 2019 15:21:16 +0200
Duration:       65s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
           job-name=pi
  Containers:
   pi:
    Image:      perl
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  14m   job-controller  Created pod: pi-5rwd7
```

可以使用`kubectl get pods`查看Job中已经完成的Pod。

如果想以可处理的方式列出Job中所有的Pod，可以这样弄：

```shell script
pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath='{.items[*].metadata.name}')
echo $pods
```

```text
pi-5rwd7
```

这里的selector跟Job的selector是一样的。`--output=jsonpath`是让返回结果中只包含每个Pod的名字。

查看Pod的标准输出：

```shell script
kubectl logs $pods
```

输出如下：

```text
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901
```

## 编写Job定义

和其他的k8s配置差不多，Job也有`apiVersion`、`kind`和`metadata`。它的名字同样也必须是一个有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

Job也要有一个[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)。

### Pod模板

`.spec`中唯一的必填字段就是`.spec.template`。

`.spec.template`就是一个[Pod模板](../../业务组件/泡德（Pod）/概要.md#Pod模板)。它跟[Pod](../../业务组件/泡德（Pod）/Pod.md)的格式是一样的，只不过它是嵌在别的属性里，没有`apiVersion`和`kind`。

除了Pod的一些必须的字段外，Job的Pod模板还需要设置好合适的标签（见[Pod选择器](#Pod选择器)）和重启策略。

[`RestartPolicy`](../../业务组件/泡德（Pod）/Pod生命周期.md#重启策略)的可选值包括`Never`、`OnFailure`。

### Pod选择器

`.spec.selector`字段是可选的。绝大部分情况都不需要你来指定。可以看看[制定你自己的Pod选择器](#制定你自己的Pod选择器)。

### 并行Job

主要由三种适合用Job来运行的任务：

1.非并行Job

- 通常，只启动一个Pod，除非Pod挂了。
- Pod成功运行完成之后Job也就随之完成了。

2. 并行Job，并带有*固定的完成数量*：

- 给`.spec.completions`设置一个非零正数。
- Job代表了整体的任务，当从1到`.spec.completions`中的每一个Pod都成功完成，Job才算完成。
- **尚未实现**：为每个Pod设置一个从1到`.spec.completions`之间的索引。

3. 并行Job，并带有*工作队列*：

- 不指定`.spec.completions`，默认为`.spec.parallelism`。
- Pod要么是相互协调，要么是由外部服务来决定每个Pod要干什么。比如某个Pod要从工作队列中提取最多N个数据项。
- 每个Pod都可以独立判定自己的工作是否都弄完了，然后整个Job也就成功完成了。
- 如果Pod以成功状态结束，不会创建新的Pod。
- 当所有Pod都结束，并且至少有一个Pod是以成功状态结束，那Job就是成功完成了。
- 当某个Pod以成功状态结束了，就不应该再有其他Pod继续工作或继续输出内容了。它们都应该处于正在退出的状态。

对于*非并行*Job，可以不用设置`.spec.completions`和`.spec.parallelism`。当都没有设置时，默认都是1。

对于*带固定完成数量*的Job，需要把`.spec.completions`设置为需要的完成数量。此时也可以设置`.spec.parallelism`，或者不设置，默认为1。

对于*带工作队列*的Job，绝对不能设置`.spec.completions`，并且将`.spec.parallelism`设置成非负整数。

对于各种Job的更多用法，可以去看[Job设计模式](#Job设计模式)。

#### 并行度控制

请求并行度（`.spec.parallelism`）可以设置成任意非负值。如果不设置，默认为1。如果设成0，Job就暂停了，直到这个值增加。

真实并行度（某一时刻运行中的Pod数量）可能会大于或小于请求并行度，原因有以下几点：

- 对于*带有固定的完成数量*的Job，真正并行运行的Pod数量不会超过剩余要完成的数量。调高`.spec.parallelism`并没有什么卵用。
- 对于*带有工作队列*的Job，只要某个Pod正常结束了就不会再启动新的Pod了——但仍允许剩余的Pod一直运行到完成状态。
- Job[控制器](../../集群架构/控制器.md)反应不过来了。
- Job无法创建Pod（`ResourceQuota`不足或权限不够等），Pod数量就可能低于请求的数量。
- 如果一大片Pod都失败了，Job控制器会降低新Pod的创建速度。
- Pod的优雅停止需要一定时间。

## Pod和容器的错误处理

Pod中的容器可能会因为各种原因出现异常，比如仅仅时由于内部进程退出码非零，或者由于OOM被杀了等等。此时，如果`.spec.template.spec.restartPolicy = "OnFailure"`，Pod还会留在原来的节点上，容器会重新运行。所以你的程序就需要处理这种本地重新运行的场景，或者设置`.spec.template.spec.restartPolicy = "Never"`。关于`restartPolicy`可以去看[Pod生命周期](../../业务组件/泡德（Pod）/Pod生命周期.md#各种状态举例)。

整个Pod也可能由于各种原因出现问题，比如Pod从节点上被踢出去了（节点升级、重启、删除等），或者内部容器异常并且`.spec.template.spec.restartPolicy = "Never"`。当Pod失败时，Job控制器会启动一个新的Pod。这就是说你的程序得处理在新Pod中重启的场景。特别是要处理上一次运行时的各种临时文件、锁、不完整的输出等等。

因此，即便你设置`.spec.parallelism = 1`、`.spec.completions = 1`，并且`.spec.template.spec.restartPolicy = "Never"`，同样的程序依然有可能被启动两次。

如果你的`.spec.parallelism`和`.spec.completions`都大于1，就可能有多个Pod一起运行。所以你的Pod还要能够处理并发的问题。

### Pod的失败补偿策略

有的时候，比如由于配置上的逻辑错误，在多次重试依然无解之后，你可能希望你的Job按照失败结束就可以了。这种情况下可以设置`.spec.backoffLimit`，定义Job失败前的重试次数。这个补偿值默认为6。失败的Pod会以指数退避来重建（10秒，20秒，40秒……），间隔6分钟。在Job的下一次检测前，如果没有出现新的错误Pod，则重置补偿计数。

> **注意**：在1.12版本前依然存在问题[#54870](https://github.com/kubernetes/kubernetes/issues/54870)

> **注意**：如果Job的`restartPolicy = "OnFailure"`，此时一定要记住，当补偿计数到达上限时，会立即停掉Job下的容器。这可能会让Job的调试非常困难。我们建议在调试Job时设置`restartPolicy = "Never"`，或者使用额外的日志系统，保证错误的Job输出不会突然丢失。

## Job的停止与清理

Job完成后就不会再创建Pod了，但也不会删除它们。之所以留着它们的狗命，就是为了让你看看日志啥的，比如错误、警告或者其他调试信息。Job对象本身同样也是不会被删除，这样你还可以继续看看它的状态信息啥的。旧的Job要由用户决定是否删除。可以用`kubectl`（比如`kubectl delete jobs/pi`或`kubectl delete -f ./job.yaml`）来删除Job。当Job删除后，它的Pod也就随之灰飞烟灭了。

默认情况下Job都是一路向西狂奔不止的，除非Pod异常（`restartPolicy=Never`）或者容器异常退出（`restartPolicy=OnFailure`），这时候Job就会去参考上面说过的`.spec.backoffLimit`了。一旦到达`.spec.backoffLimit`Job就会被标记成失败并且停止所有运行中的Pod。

还有一种停止Job的方法是设置一个活动状态期限。也就为`.spec.activeDeadlineSeconds`设置合适的秒数。`activeDeadlineSeconds`控制着Job的持续时间，跟它创建了多少Pod没有关系。当Job到达`activeDeadlineSeconds`，会停止所有运行中的Pod，Job状态标记为`type: Failed`，以及`reason: DeadlineExceeded`。

这里注意一下，`.spec.activeDeadlineSeconds`的优先级要高于`.spec.backoffLimit`。因此，当到达`activeDeadlineSeconds`时，Job就不会继续对异常的Pod进行重试操作了，即便还没有达到`backoffLimit`。

栗子：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

这里同样也要注意一下，Job的Spec和[Pod模板的Spec](../泡德（Pod）/初始化容器.md#细节)中都有`activeDeadlineSeconds`。设置的时候一定要放到正确的层级中。

另外需要牢记的是，`restartPolicy`是用在Pod上的，不是Job的：一旦Job的状态变成`type: Failed`，是不存在什么自动重启的机制的。这就是说，如果是因为触发了`.spec.activeDeadlineSeconds`和`.spec.backoffLimit`，Job就会永久性的停止，必须人工干预进行处理。

## 自动清理已完成的Job

已完成的Job一般都没有什么存在的价值了。留着它们的狗命只会为apiserver添加负担。如果Job是由更高层的控制器直接管理，比如[CronJob](CronJob.md)，CronJob会基于容量策略来清理Job。

### TTL机制

**功能状态**：`Kubernetes v1.12 [alpha]`

另一种自动清理Job的方法是使用[TTL控制器](TTL控制器.md)，也就是设置Job的`.spec.ttlSecondsAfterFinished`字段。

当TTL控制器对Job进行清理时，它会对Job进行级联删除，比如删除它的依赖对象，比如Job中的各种Pod。Job被删除的时候，各种finalizer依然会按照规章制度来执行。

栗子：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

在`pi-with-ttl`这个Job结束`100`秒后，就会被自动删除了。

如果将这个字段设置为`0`，Job结束后就会立即被删除。如果不设置该字段，也就不会被自动删除了。

注意目前这个TTL机制还是alpha阶段，要使用特性门`TTLAfterFinished`。可以学习[TTL控制器](TTL控制器.md)，了解更多新闻资讯。

## Job设计模式

Job可以用来实现可靠的Pod并行执行机制。Job不是为那种紧密联系的并行进程而设计，比如常见的科学计算。它的意义更多是在于并行处理一组相互独立但又有些关系的*工件*。比如要发送一大堆email，要渲染的一组框架，要转码的一堆文件，或者是对NoSQL中某个范围的key进行扫描，等等。

在较复杂的系统中，会存在多种不同的工件集合。这里我们只考虑它们中的某一组，用户想要一起处理——*批处理任务*。

关于并行计算有这么几种不同的模式，各有利弊。权衡之处在于：

- 每个工件一个任务VS一个任务处理所有工件。对于工件数量庞大的场景，肯定是后者更好。前者增加了用户和系统的负担，弄出来一大堆Job对象。
- Pod数量等于工件数量VS每个Pod可以处理多个工件。前者通常不怎么需要修改现有代码和容器。后者用于工件数量更大的场景，原因跟上一条是一样的。
- 基于工作队列。这种场景需要一个队列服务，需要修改已有程序或容器，让它们走队列。上面的方法更适合利用那些已存在的容器应用。

关于这几条我们总结到了下面，其中2~4列对应上面列出的权衡场景。每个模式的名字可以点击直接跳到相关的栗子上，有更详细的内容。

模式|单个Job对象|Pod数量少于工件数量？|使用已有的程序？|能用在Kube 1.1中？
-|-|-|-|-
[Job模板扩展]()|||√|√
[队列，每个工件一个Pod]()|√||偶尔|√
[队列，Pod数量不定]()|√|√||√
单一Job且任务固定|√||√|

当你设置了`.spec.completions`，Job控制器创建的每个Pod都具有相同的[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)。这就是说为一个任务创建的所有Pod都有相同的命令行内容和相同的镜像、数据卷，以及（几乎）相同的环境变量。这些模式就是为不同的类型的工作来构建不同的Pod组织方式。

下表针对每种模式列出了`.spec.parallelism`和`.spec.completions`的设置方式。这里的`W`代表工件的数量。

模式|`.spec.completions`|`.spec.parallelism`
-|-|-
[Job模板扩展]()|1|必须是1
[队列，每个工件一个Pod]()|W|任意
[队列，Pod数量不定]()|1|任意
单一Job且任务固定|W|任意

### 制定你自己的Pod选择器