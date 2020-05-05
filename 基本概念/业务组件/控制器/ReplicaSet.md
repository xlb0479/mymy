# ReplicaSet

一个ReplicaSet就是要在任何时候为Pod维护住一个稳定的副本集合。因此，常用它来保证Pod可用的副本数量。

- [它如何工作](#它如何工作)
- [啥时候用](#啥时候用)
- [栗子](#栗子)
- [捕获无模板Pod](#捕获无模板Pod)
- [编写ReplicaSet](#编写ReplicaSet)
- [各种操作](#各种操作)
- [其他方案](#其他方案)

## 它如何工作

ReplicaSet中的字段包括：用于定义哪些Pod属于它管辖范围的选择器，定义需要维护副本的数量，以及副本不够的时候用于创建Pod的Pod模板。ReplicaSet通过增删Pod来满足设定的目标数量。需要新增的时候，就用它的Pod模板。

ReplicaSet跟它的Pod通过Pod的[metadata.ownerReferences]()建立关系，这玩意可以标明当前对象归属于哪种资源。每个被俘虏的Pod都有这种标明主人信息的ownerReferences字段。ReplicaSet通过这条纽带可以得知Pod的状态，据此实施自己的小阴谋。

ReplicaSet通过自身的选择器来标明新的Pod。如果一个Pod没有OwnerReference或者OwnerReference指向的并不是一个[控制器](../../集群架构/控制器.md)，但是匹配到了ReplicaSet的选择器，那瞬间就会被ReplicaSet俘虏。

## 啥时候用

ReplicaSet可以确保任何时刻都有指定数量的Pod副本在运行。但是，重点来了，Deployment是一个相对更高级的概念，它能控制ReplicaSet，还能提供一些声明式的更新策略以及其他好东西。因此，我们建议你去用Deployment，而不是直接闹ReplicaSet，除非你是准备自己安排更新策略，或者压根不需要更新。

这就意味着你这辈子可能不需要直接操作ReplicaSet对象：去用Deployment吧，在spec中定义好你的应用。

## 栗子

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

保存到`frontend.yaml`文件并提交到k8s。

```shell script
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

获取当前部署的ReplicaSet：

```shell script
kubectl get rs
```

看到你创建的前端应用：

```text
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       6s
```

检查ReplicaSet的状态：

```shell script
kubectl describe rs/frontend
```

得到下面这样的输出：

```text
Name:         frontend
Namespace:    default
Selector:     tier=frontend
Labels:       app=guestbook
              tier=frontend
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"labels":{"app":"guestbook","tier":"frontend"},"name":"frontend",...
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  tier=frontend
  Containers:
   php-redis:
    Image:        gcr.io/google_samples/gb-frontend:v3
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  117s  replicaset-controller  Created pod: frontend-wtsmm
  Normal  SuccessfulCreate  116s  replicaset-controller  Created pod: frontend-b2zdv
  Normal  SuccessfulCreate  116s  replicaset-controller  Created pod: frontend-vcmts
```

检查Pod信息：

```shell script
kubectl get pods
```

得到如下输出：

```text
NAME             READY   STATUS    RESTARTS   AGE
frontend-b2zdv   1/1     Running   0          6m36s
frontend-vcmts   1/1     Running   0          6m36s
frontend-wtsmm   1/1     Running   0          6m36s
```

检查Pod的归属信息：

```shell script
kubectl get pods frontend-b2zdv -o yaml
```

得到如下输出，metadata中有ownerReferences字段：

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-02-12T07:06:16Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-b2zdv
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: frontend
    uid: f391f6db-bb9b-4c09-ae74-6a1f77f3d5cf
...
```

## 捕获无模板Pod

如果你已经建了一些果体Pod（bare pod），强烈建议这些果体Pod的标签不要匹配到ReplicaSet的选择器上。因为ReplicaSet不光可以捕获它模板里的Pod——如前文所说，是通过选择器捕获的。

接着上一个例子，此时加入下面的Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    tier: frontend
spec:
  containers:
  - name: hello1
    image: gcr.io/google-samples/hello-app:2.0

---

apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    tier: frontend
spec:
  containers:
  - name: hello2
    image: gcr.io/google-samples/hello-app:1.0
```

这些Pod没有归属资源，但是匹配了ReplicaSet的选择器，会被瞬间捕获。

假设我们之前创建的ReplicaSet以及它定义的Pod都部署完了，这个时候我们再去创建这两个新的Pod：

```shell script
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

新的Pod会被ReplicaSet瞬间捕获，紧接着立即被停掉了，因为ReplicaSet发现副本数超出了预设值。

查看Pod：

```shell script
kubectl get pods
```

返回的结果表明，新的Pod要么已经被停掉了，要么是正在停止中：

```text
NAME             READY   STATUS        RESTARTS   AGE
frontend-b2zdv   1/1     Running       0          10m
frontend-vcmts   1/1     Running       0          10m
frontend-wtsmm   1/1     Running       0          10m
pod1             0/1     Terminating   0          1s
pod2             0/1     Terminating   0          1s
```

相反，如果我们在ReplicaSet之前先建好Pod：

```shell script
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

然后再去建ReplicaSet：

```shell script
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

你会发现ReplicaSet捕获了已经存在的两个Pod，然后只新建了一个Pod就能满足它的副本数要求了。查看Pod信息：

```shell script
kubectl get pods
```

得到如下输出：

```text
NAME             READY   STATUS    RESTARTS   AGE
frontend-hmmj2   1/1     Running   0          9s
pod1             1/1     Running   0          36s
pod2             1/1     Running   0          36s
```

如此说来，ReplicaSet是可以非亲生的Pod的。

## 编写ReplicaSet

跟其他API对象一样，ReplicaSet也有`apiVersion`、`kind`和`metadata`。其中kind对于ReplicaSet来说就写死ReplicaSet就行了。从1.9版本开始，最新的API version是`apps/v1`，默认就是它。`apps/v1beta2`已经废弃了。可以看一下前面例子中的`frontend.yaml`。

ReplicaSet对象的名字必须是有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

再就是[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)。

### Pod模板

`.spec.template`就是[Pod模板](../泡德（Pod）/概要.md#Pod模板)，里面必须要包含标签信息。在我们的`frontend.yaml`例子中有一个标签：`tier: frontend`。注意标签不要匹配到其他控制器上，避免它们过来“爱抚”这个Pod。

对于模板的[重启策略](../泡德（Pod）/Pod生命周期.md#重启策略)字段，`.spec.template.spec.restartPolicy`，唯一允许的值是`Always`，也是默认值。

### Pod选择器

`.spec.selector`字段定义了[标签选择器](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)。如[前文所述](#它如何工作)，这里的标签是用来捕获潜在的Pod。在`frontend.yaml`中，我们是这么写的：

```yaml
matchLabels:
    tier: frontend
```

对于ReplicaSet，它的`.spec.template.metadata.labels`必须跟`spec.selector`匹配，否则API会报错。

>**注意**：如果有两个`.spec.selector`相同的ReplicaSet，但是`.spec.template.metadata.labels`跟`.spec.template.spec`都不相同，每个ReplicaSet会忽略另一个ReplicaSet创建的Pod。

### Replicas

用`.spec.replicas`字段定义Pod的并发数量。ReplicaSet会通过增/删Pod来匹配这个数量。

如果没有定义`.spec.replicas`，默认是1。

## 各种操作

### 删除ReplicaSet及其Pod

要删除一个ReplicaSet及其所有Pod，请用[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)。[垃圾回收器]()会默认自动删除所有相关的Pod。

如果是用REST API或者`client-go`库，必须在-d选项中把`propagationPolicy`设置为`Background`或`Foreground`。例如：

```shell script
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
> -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
> -H "Content-Type: application/json"
```

### 只删除ReplicaSet

使用`kubectl delete`(https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)的时候，可以设置`--cascade=false`，这样就可以只删ReplicaSet而不删它的Pod了。如果是用REST API或者`client-go`库，必须把`propagationPolicy`设置为`Orphan`。例如：

```shell script
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
> -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
> -H "Content-Type: application/json"
```

删了ReplicaSet之后，可以建一个新的ReplicaSet来替代它。只要`.spec.selector`不变，之前的Pod还可以被正常捕获。但是，它不会去修改Pod让它满足新的Pod模板。如果要以可控的方式更新Pod的定义，那你就用[Deployment]()，因为ReplicaSet不直接支持滚动更新。

### 将Pod从ReplicaSet中分离

可以通过修改标签，让Pod从ReplicaSet中分离出去。这种高新科技可以用在将Pod从某个服务中删除，然后进行调试或数据恢复等操作。用这种方式被分离的Pod会被自动替代（假设我们没有修改副本数要求）。

### 扩展ReplicaSet

通过更新`.spec.replicas`字段可以轻而易举地对ReplicaSet的副本规模进行变更。ReplicaSet会确保Pod数量达到要求。

### 将ReplicaSet作为Horizontal Pod Autoscaler的目标

ReplicaSet可以作为[Horizontal Pod Autoscaler（HPA）]()的目标。就是说HPA可以控制ReplicaSet来自主伸缩。下面就是一个HPA的例子，可以控制前文例子中的ReplicaSet。

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

保存到`hpa-rs.yaml`文件并提交到k8s，然后就能创建一个根据CPU使用率来自动伸缩ReplicaSet的HPA了。

```shell script
kubectl apply -f https://k8s.io/examples/controllers/hpa-rs.yaml
```

此外，还可以用`kubectl autoscale`命令实现同样的效果（更简单哦！）

```shell script
kubectl autoscale rs frontend --max=10
```

## 其他方案

### Deployment（建议）

[`Deployment`]()这玩意可以包含ReplicaSet，并且可以通过声明式的、在服务端一侧的滚动更新对ReplicaSet和它的Pod进行更新。虽然ReplicaSet可以单独使用，但是在现在这个年月，它们实际上主要都是辅助Deployment对Pod进行创建、删除和更新。使用Deployment的时候可以不用操心由它产生的ReplicaSet。Deployment会自己照顾自己的ReplicaSet。因此当你需要用ReplicaSet的时候，我们建议用Deployment。

### 果Pod（Bare Pod）

ReplicaSet不需要用户直接去创建Pod，而是由它自己管理，比如节点闹妖，或者内核升级维护的时候将节点分离，此时它都可以自动删除或停止Pod，然后自动创建新的Pod来替代。正因如此，即便你的应用只需要一个Pod我们也建议你用ReplicaSet。这跟进程的Supervisor有点类似，只不过这里管理的是多个节点上的Pod，而不是某个节点上的进程。ReplicaSet会将本地容器的重启工作委托到节点的某个代理上（比如Kubelet或Docker）。

### Job

如果Pod需要在完成任务后自动停止，那就用[`Job`]()替代ReplicaSet。

### DaemonSet

如果需要Pod提供机器层面的功能，比如服务器监控或者日志记录，那就应该用[`DaemonSet`]()。这些Pod的命运是跟底层机器绑定的：在其他类型的Pod启动之前就要先启动这种类型的Pod，服务器要重启/关机的时候可以安全的停止这些Pod。

### ReplicationController

先有的[*ReplicationController*]()，后有的ReplicaSet。它俩的目的是一样的，行为也差不多，只不过ReplicationController不支持用条件集合来选定[标签](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)。所以，同样的情况下现在基本上都用ReplicaSet了。