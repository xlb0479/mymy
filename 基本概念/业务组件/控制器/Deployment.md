# Deployment

*Deployment*可以声明式地更新[Pod](../泡德（Pod）/Pod.md)和[ReplicaSet](ReplicaSet.md)。

我们在Deployment中定义好*目标状态（desired state）*，然后Deployment[控制器](../../集群架构/控制器.md)就会以某种受控的速率将真实状态向目标状态推进。可以用Deployment创建新的ReplicaSet，或者删除已经存在的Deployment，把它的所有遗产交给一个新的Deployment。

>**注意**：不要直接去操作由Deployment管理的ReplicaSet。如果你遇到的场景没有包含在下面的列表中，你去k8s开个issue说一下吧。

- [用例](#用例)
- [创建](#创建)
- [更新](#更新)
- [回滚](#回滚)
- [伸缩](#伸缩)
- [暂停与恢复](#暂停与恢复)
- [各种状态](#各种状态)
- [清理机制](#清理机制)
- [灰度发布](#灰度发布)
- [编写Spec](#编写Spec)

## 用例

下面是一些Deployment的典型用例：

- [用Deployment来滚动发布ReplicaSet](#创建)。由ReplicaSet在后台创建Pod。检查状态以确认发布进展。
- 更新Deployment的PodTemplateSpec来[声明新的Pod状态](#更新)。会创建新的ReplicaSet，Deployment以受控的速率将Pod从旧的ReplicaSet迁移到新的下面。每一个新的ReplicaSet都会更新Deployment的版本。
- 如果Deployment当前状态有问题那就[回滚到上一个版本](#回滚)。每次回滚都会更新Deployment的版本。
- [对Deployment进行伸缩以适应负载变化](#伸缩)。
- [暂停Deployment](#暂停与恢复)，对PodTemplateSpec进行多处更新，然后恢复并进行一个新的滚动发布。
- 通过[Deployment的各种状态](#各种状态)分析发布进程是否卡住了。
- [清理不需要的ReplicaSet](#清理机制)。

## 创建

下面是一个Deployment的例子。它会创建一个ReplicaSet，孕育着三个`nginx`Pod：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

在本例中：

- 建了一个叫`nginx-deployment`的Deployment，由`.metadata.name`字段定义。
- Deployment建了三个Pod副本，由`.spec.replicas`字段定义。
- `.spec.selector`字段定义了Deployment搜寻俘虏Pod的方式。本例中只是选择了一个定义在Pod模板中的标签（`app: nginx`）。只要Pod模板满足条件，可以实现各种负载的选择方式。

>**注意**：`.spec.selector.matchLabels`字段是一个{key,value}对组成的map。`matchLabels`里的每一个键值对都等同于`matchExpressions`中的一个元素，key就是“key”，operator就是“In”，value数组就只包含“value”。`matchLabels`和`matchExpressions`中的条件必须全部满足才能匹配。

- `template`字段包含了一些子属性：
    - Pod的标签通过`.metadata.labels`定义为`app: nginx`。
    - Pod模板声明，或者直接说`.template.spec`字段，表明Pod运行一个`nginx`容器，镜像来自于[Docker Hub](https://hub.docker.com/)的`nginx`镜像，镜像版本1.14.2。
    - 通过`.template.spec.containers[0].name`字段定义容器名为`nginx`。

参照下面的步骤来创建这个Deployment：

开始之前，先确认你的k8s集群已经正常运行起来了，不然就搞笑了。

- 1. 使用下面的命令创建Deployment：

>**注意**：可以加上`--record`选项，这样就会把这条命令本身记录到资源的`kubernetes.io/change-cause`注解中。这样可以便于后续的问题分析，比如查看每个Deployment版本执行的命令。

```shell script
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

- 2.执行`kubectl get deployments`检查是否创建完成。如果Deployment正处于创建中，命令返回的内容就类似下面这样：

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```

查看Deployment时会显示以下字段：

- `NAME`Deployment的名字。
- `READY`显示当前可用的副本数量。格式为就绪/总数。
- `UP-TO-DATE`显示已经达到目标状态的副本数量。
- `AVAILABLE`显示用户当前可用的副本数量。
- `AGE`显示应用持续运行的时间。

注意这里面的副本总数对应的就是`.spec.replicas`字段的值。

- 3.检查Deployment的滚动发布状态，执行`kubectl rollout status deployment.v1.apps/nginx-deployment`。输出如下：

```text
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

- 4.过一会儿再次执行`kubectl get deployments`。输出如下：

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           18s
```

注意此时Deployment已经创建了全部的三个副本，所有副本都已是up-to-date（已经实现了最新的Pod模板）并且可用。

- 5.执行`kubectl get rs`查看由Deployment创建的ReplicaSet（`rs`）。输出如下：

```text
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
```

ReplicaSet显示如下字段：

- `NAME`ReplicaSet的名字。
- `DESIRED`应用的目标*副本数*，创建Deployment时已定义。此之谓*目标*状态。
- `CURRENT`显示当前运行的副本数。
- `READY`显示已经就绪的副本数。
- `AGE`显示应用持续运行的时间。

注意ReplicaSet的格式是`[DEPLOYMENT名]-[随机字符串]`。其中的随机字符串是以`pod-template-hash`做为seed来生成的。

- 6.执行`kubectl get pods --show-labels`查看每个Pod的标签。输出如下：

```text
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

ReplicaSet来确保一共有三个`nginx`Pod。

>**注意**：必须在Deployment中定义好合适的选择器和Pod标签（比如例子中的`app: nginx`）。不要跟其他控制器（包括其他Deployment和StatefulSet）的标签、选择器范围产生重叠。k8s不会阻止这种重叠声明，如果出现这种情况，控制器可能会出现冲突，产生不可预料的行为。

### Pod-template-hash标签

>**注意**：别改这个标签。

Deployment会给它创建、控制的ReplicaSet都打上`pod-template-hash`标签。

这个标签可以用来确保Deployment的ReplicaSet不会出现重叠。这个值是对`PodTemplate`进行hash后将结果作为标签值添加到ReplicaSet的选择器、Pod模板标签以及ReplicaSet下已经存在的Pod上。

## 更新

>**注意**：只有当Pod模板（`.spec.template`）发生改变才会触发Deployment的滚动发布，比如修改模板中的标签或者容器镜像。其他的，比如对Deployment进行伸缩，不会触发滚动发布。

参照下面的步骤来更新Deployment：

- 1.将nginx的镜像版本从`1.14.2`更新到`1.16.1`。

```shell script
kubectl --record deployment.apps/nginx-deployment set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

可以简化为：

```shell script
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
```

返回结果：

```text
deployment.apps/nginx-deployment image updated
```

或者，还可以`edit`Deployment，将`.spec.template.spec.container[0].image`的值从`nginx:1.14.2`改为`nginx:1.16.1`：

```shell script
kubectl edit deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployment.apps/nginx-deployment edited
```

- 2.查看滚动发布的状态：

```shell script
kubectl rollout status deployment.v1.apps/nginx-deployment
```

返回结果：

```text
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```

或：

```text
deployment.apps/nginx-deployment successfully rolled out
```

查看Deployment更新后的各种信息：

- 滚动发布成功之后，执行`kubectl get deployments`查看Deployment信息。返回结果如下：

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           36s
```

- 执行`kubectl get rs`发现Deployment创建了一个新的ReplicaSet，扩展到3个副本，并将旧的缩减到0个副本。

```shell script
kubectl get rs
```

返回结果：

```text
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```

- 执行`get pods`，此时只会看到最新的Pod：

```shell script
kubectl get pods
```

返回结果：

```text
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

以后再想更新的话，修改Deployment的Pod模板就行了。

Deployment会确保在更新过程中只有一部分Pod会停掉。默认情况下，会确保目标数量75%的Pod处于运行中的状态（最多停掉25%）。

Deployment还会确保在更新过程中创建的Pod数量保持在一定比例。默认情况下最多达到目标数量的125%（最多超出25%）。

比如，你仔细看上面的栗子，会发现它先是建一个新的Pod，然后删除某个旧的，然后再继续创建新的。直到有足够数量的新Pod启动才会去杀掉旧的Pod，同样，直到足够数量的旧Pod被杀掉才会继续创建新的Pod。确保整个过程中最少有2个，最多4个Pod可用。

- 查看Deployment详情：

```shell script
kubectl describe deployments
```

返回结果：

```text
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
Labels:  app=nginx
 Containers:
  nginx:
    Image:        nginx:1.16.1
    Port:         80/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

如果你是个有心人，你会发现，第一次创建Deployment的时候，它建了一个ReplicaSet（nginx-deployment-2035384211）然后直接扩展到3个副本，一点儿面子不给。当你更新Deployment后，它先是建了个新的ReplicaSet（nginx-deployment-1564180365）并扩充到1个副本，然后把旧的ReplicaSet缩减到2个副本，所以整个过程中最少2个最多4个Pod可用。它使用同样的滚动更新策略，持续地对新旧ReplicaSet进行扩充和缩减。最后，新的ReplicaSet下面有3个可用的副本，旧的缩减成0。

### Rollover（多个更新同步进行）

每当Deployment控制器发现一个新的Deployment，就要建一个ReplicaSet然后推到目标副本数。如果Deployment更新，旧的ReplicaSet就要开始缩减。最终，新的ReplicaSet扩充到`.spec.replicas`，旧的ReplicaSet缩减到0。

如果一个Deployment的滚动发布正在进行中，此时你又更新了Deployment，每次Deployment更新都会创建一个新的ReplicaSet然后开始扩充，如果当前已经有正在扩充的ReplicaSet了，会进行覆盖——会将前一个正在扩充的ReplicaSet也作为旧的ReplicaSet并开始对它进行缩减。

比如说，你建了一个5副本的`nginx:1.14.2`，然后把它更新成`nginx:1.16.1`，而此时只创建了3个`nginx:1.14.2`。Deployment会立即开始杀掉已经建好的3个`nginx:1.14.2`，并开始创建`nginx:1.16.1`。它不会等5个`nginx:1.14.2`都建完了再进行更新。

### 修改标签选择器

一般不建议修改标签选择器，而且这种东西应该都是提前规划好了的。如果你真的需要改一下，你首先得变成一个胆大心细的人，并且对整个过程中可能发生的任何事都有充分的了解。

>**注意**：在API版本`apps/v1`中，Deployment的标签选择器建了之后就不能改了。

- 如果是添加新的选择，那就要同时修改Pod模板中的标签，否则会校验失败。这种修改不会发生覆盖，也就是说，新的选择器不会选择之前的ReplicaSet和Pod，会导致旧的ReplicaSet孤苦伶仃地度过下半生。
- 修改某个选择器的value——跟上面的结果一样。
- 删除某个选择器的key——不需要修改Pod模板中的标签。已存在的ReplicaSet不会孤独终老，而且也不会创建新的ReplicaSet，但是，注意一下，虽然选择器里删掉了，但是这个标签在ReplicaSet和Pod上依然存在。

## 回滚

有的时候你需要回滚Deployment；比如Deployment出问题了，死循环啥的。默认情况下，Deployment的所有发布历史都会保存在系统中，可以随时回滚（可以修改版本历史记录数量上限）。

>**注意**：每当触发Deployment的滚动发布时就会创建一个新的版本记录。这就是说，当且仅当Deployment的Pod模板（`.spec.template`）发生变化的时候才会创建版本记录，比如修改了模板的标签或容器镜像。其他更新操作，比如对Deployment进行伸缩，不会创建新的版本，可以同时进行手动/自动的伸缩操作。所以，当你回滚时，只有Pod模板发生了回滚。

- 比如更新Deployment的时候打错字了，本该是`nginx:1.16.1`结果你打成`nginx:1.161`了：

```shell script
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.161 --record=true
```

返回结果：

```text
deployment.apps/nginx-deployment image updated
```

- 滚动发布卡住了。可以用命令看到：

```shell script
kubectl rollout status deployment.v1.apps/nginx-deployment
```

返回结果：

```text
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

- 按下Ctrl-C停止状态监控。关于卡住的更多信息，[看这儿](#各种状态)。
- 可以看到旧的副本有2个（`nginx-deployment-1564180365`和`nginx-deployment-2035384211`），新的副本有1个（`nginx-deployment-3066724191`）。

```shell script
kubectl get rs
```

返回结果：

```text
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   1         1         0       6s
```

- 看一下Pod，发现新ReplicaSet建了1个Pod，一直在那里无限拉镜像。

```shell script
kubectl get pods
```

返回结果：

```text
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-1564180365-hysrc   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
```

>**注意**：Deployment控制器可以自动停止有问题的滚动发布，停止扩充新的ReplicaSet。具体取决于rollingUpdate参数（`maxUnavailable`）。默认为25%。

- 查看Deployment：

```shell script
kubectl describe deployment
```

返回结果：

```text
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 desired | 1 updated | 4 total | 3 available | 1 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.161
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:     nginx-deployment-1564180365 (3/3 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (1/1 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubObjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 1
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
```

要想让这个世界恢复平静，需要将Deployment回滚到上一个稳定的版本。

### 检查Deployment的发布历史

按照以下步骤检查发布历史：

- 1.查看Deployment的版本历史：

```shell script
kubectl rollout history deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml --record=true
2           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
3           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.161 --record=true
```

`CHANGE-CAUSE`是在创建过程中从Deployment的`kubernetes.io/change-cause`注解中拷贝到版本信息中的。   
可以使用下面的方式指定`CHANGE-CAUSE`内容：

- 给Deployment设置注解

```shell script
kubectl annotate deployment.v1.apps/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
```

- 给`kubectl`命令增加`--record`选项可以将命令内容保存到资源信息中。
- 手动编辑资源信息。

- 2.查看每个版本的详情：

```shell script
kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2
```

返回结果：

```text
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### 回滚到之前的版本

参照下面的步骤可以将Deployment回滚到版本2。

- 1.现在，你痛定思痛，决定回滚到上一个版本：

```shell script
kubectl rollout undo deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployment.apps/nginx-deployment rolled back
```

你还可以用`--to-revision`回滚到指定的版本：

```shell script
kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
```

返回结果：

```text
deployment.apps/nginx-deployment rolled back
```

关于rollout相关的命令可以看一下[`kubectl rollout`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout)。

现在Deployment回滚到上一个版本了。你会看到Deployment为其生成了一个`DeploymentRollback`事件。

- 2.检查回滚是否成功，Deployment是否正常：

```shell script
kubectl get deployment nginx-deployment
```

返回结果：

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30m
```

- 3.查看Deployment详情：

```shell script
kubectl describe deployment nginx-deployment
```

返回结果：

```text
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 02 Sep 2018 18:17:55 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=4
                        kubernetes.io/change-cause=kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.16.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-c4747d96c (3/3 replicas created)
Events:
  Type    Reason              Age   From                   Message
  ----    ------              ----  ----                   -------
  Normal  ScalingReplicaSet   12m   deployment-controller  Scaled up replica set nginx-deployment-75675f5897 to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 0
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-595696685f to 1
  Normal  DeploymentRollback  15s   deployment-controller  Rolled back deployment "nginx-deployment" to revision 2
  Normal  ScalingReplicaSet   15s   deployment-controller  Scaled down replica set nginx-deployment-595696685f to 0
```

## 伸缩

执行如下操作，对Deployment进行伸缩：

```shell script
kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
```

返回结果：

```text
deployment.apps/nginx-deployment scaled
```

如果集群开启了[Pod自动伸缩功能]()，你可以给Deployment设置一个自动伸缩器（autoscaler），基于CPU使用率设置好Pod数量的最大最小值。

```shell script
kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

返回结果：

```text
deployment.apps/nginx-deployment scaled
```

### 均匀伸缩

Deployment的滚动更新可以在某一时刻同时运行应用的多个版本。当某一次滚动发布（rollout）正在进行中（进行中或被暂停），此时你或者自动伸缩起又触发了一次Deployment的滚动更新，为了降低风险，Deployment控制器会平衡两个版本的ReplicaSet（和它的Pod）。此之谓*均匀伸缩（proportional scaling）*。

比如当前的Deployment有10个副本，[maxSurge](#max-surge)=3，[maxUnavailable](#max-unavailable)=2。

- 确认10个副本都在运行。

```shell script
kubectl get deploy
```

返回结果：

```text
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

- 故意把镜像改成满世界都找不到的版本

```shell script
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:sometag
```

返回结果：

```text
deployment.apps/nginx-deployment image updated
```

- 镜像更新触发了一次新的滚动发布，ReplicaSet为nginx-deployment-1989198191，但是因为我们前面设置了`maxUnavailable`，所以它卡住了。检查滚动发布的状态：

```shell script
kubectl get rs
```

返回结果：

```text
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

- 此时会产生一个新的伸缩请求。自动伸缩器要将Deployment的副本数推到15。Deployment控制器就要决定这5个新的副本放在哪。如果没有均匀伸缩，5个新的副本全都会加到新的ReplicaSet下。如果有了均匀伸缩，嘿嘿，这些副本就会在所有ReplicaSet上分配。较多的部分分给当前副本数最多的ReplicaSet，较少的部分给当前副本较少的ReplicaSet。如果有剩下的，都分给副本数最多的ReplicaSet。副本数为零的ReplicaSet不变。

最终，旧的ReplicaSet加了3个副本，新的ReplicaSet加了2个副本。滚动发布最终要将所有的副本都推到新的ReplicaSet下面，认为新的副本都能用。确认当前的状态：

```shell script
kubectl get deploy
```

返回结果：

```text
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
```

从当前状态可以看出每个ReplicaSet加了多少副本。

```shell script
kubectl get rs
```

返回结果：

```text
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

## 暂停与恢复

在进行更新前，可以先把Deployment暂停了，然后再恢复过来。这样就可以在这期间进行多次修复，避免触发不必要的滚动发布。

- 比如刚刚建好了要给Deployment：

```shell script
kubectl get deploy
```

返回结果：

```text
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
```

查看滚动发布的状态：

```shell script
kubectl get rs
```

返回结果：

```text
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```

- 执行暂停命令：

```shell script
kubectl rollout pause deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployment.apps/nginx-deployment paused
```

- 更新镜像：

```shell script
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

返回结果：

```text
deployment.apps/nginx-deployment image updated
```

- 发现没有触发新的滚动发布：

```shell script
kubectl rollout history deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>
```

- 查看滚动发布状态，确认Deployment是否更新成功：

```shell script
kubectl get rs
```

返回结果：

```text
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```

- 可以随便进行各种更新，比如再改一下资源使用限制：

```shell script
kubectl set resources deployment.v1.apps/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```

返回结果：

```text
deployment.apps/nginx-deployment resource requirements updated
```

Deployment暂停前的那些Pod会一直正常运行，只要Deployment是暂停状态，新的更新就不会产生任何实际影响。

- 最后，恢复Deployment，此时会产生一个新的ReplicaSet，包含了之前做的所有更新操作：

```shell script
kubectl rollout resume deployment.v1.apps/nginx-deployment
```

返回结果：

```text
deployment.apps/nginx-deployment resumed
```

- 持续监控滚动发布的状态。

```shell script
kubectl get rs -w
```

返回结果：

```text
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
```

- 查看最新的状态：

```shell script
kubectl get rs
```

返回结果：

```text
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

>**注意**：暂停状态下的Deployment不能回滚，必须先恢复Deployment。

## 各种状态

Deployment一生当中会进入不同的状态。比如发布新的ReplicaSet时会进入[processing](#processing)状态，又或是[complete](#complete)，也可能不幸变成[failed](#failed)。

### Processing

当满足以下条件之一时，k8s就会把Deployment标记成*processing*：

- Deployment创建了一个新的ReplicaSet。
- Deployment正在扩充新的ReplicaSet。
- Deployment正在缩减旧的ReplicaSet。
- 新的Pod状态变为ready或available（ready状态持续超过[MinReadySeconds](#min-ready-seconds)）。

可以执行`kubectl rollout status`来监控Deployment的状态变化。

### Complete

当Deployment满足以下条件时，k8s会将Deployment标记成*complete*：

- Deployment的所有副本都已更新到最新的版本，即所有更新全部完成。
- Deployment的所有副本都进入可用（available）状态。
- Deployment没有还在运行中的旧副本。

可以执行`kubectl rollout status`检查Deployment的状态是否是complete。如果滚动发布正常完成，`kubectl rollout status`执行后的退出码为零。

```shell script
kubectl rollout status deployment.v1.apps/nginx-deployment
```

返回结果：

```text
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment.apps/nginx-deployment successfully rolled out
$ echo $?
0
```

### Failed

Deployment在部署新的ReplicaSet时可能会一直卡在那儿。这可能是因为以下几种原因导致：

- 资源配额（quota）不足
- 就绪探测（readiness probe）失败
- 镜像拉不下来（便秘）
- 权限不够
- 资源限制
- 应用配置错误

可以在Deployment的spec中设置期限参数（[`.spec.progressDeadlineSeconds`](#progress-deadline-seconds)）来快速发现这些问题。`.spec.progressDeadlineSeconds`参数用来设置经过多长时间后Deployment控制器就会认为它卡住了。

下面的`kubectl`命令设置了`progressDeadlineSeconds`，如果Deployment超过10分钟没动静，控制器就会做出响应：

```shell script
kubectl patch deployment.v1.apps/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```

返回结果：

```text
deployment.apps/nginx-deployment patched
```

到达死期后，Deployment控制器会添加一个DeploymentCondition，添加到Deployment的`.status.conditions`中，属性如下：

- Type=Progressing
- Status=False
- Reason=ProgressDeadlineExceeded

关于状态的condition，可以去看[k8s API规范](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties)。

>**注意**：除了报告一下`Reason=ProgressDeadlineExceeded`，k8s不会对卡住的Deployment有其他操作。更高级的编排器可以利用这种状态做出合适的动作，比如将Deployment回滚到上一个版本。

>**注意**：如果Deployment暂停了，此时k8s不会去检查它的死期。所以可以在滚动发布过程中安全地暂停和恢复Deployment，不用担心触发死期将至的紧迫感。

可能偶尔会出现那么一小会儿的错误状态，可能是某个超时时间设置的有点儿短，或者是其他短暂的错误。比如我们现在假设你资源配额不足了。你查看Deployment的信息就会发现：

```shell script
kubectl describe deployment nginx-deployment
```

返回结果：

```text
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

执行`kubectl get deployment nginx-deployment -o yaml`能看到如下信息：

```text
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

## 清理机制

## 灰度发布

## 编写Spec