# StatefulSets

顾名思义，见字如面，名如其意，StatefulSet就是用来管理有状态服务的API对象。

它负责[Pod](../泡德（Pod）/概要.md)的部署、伸缩，*并且为Pod提供顺序和唯一性相关的保证*。

跟[Deployment](Deployment.md)差不多，StatefulSet也是基于一份同样的容器spec来管理Pod的。不同之处在于，StatefulSet为每个Pod维护了一个严格的身份标识。这些Pod基于同样的spec创建，但是无法互换：无论重新进行多少次调度，每个Pod都维护者一个持久化的身份标识。

如果想要为业务负载提供持久的数据卷存储，可以同时把StatefulSet放到你的解决方案中。尽管每个Pod都可能出现异常情况，但是StatefulSet中的每个Pod都有持久化的身份标识，当新的Pod要替换异常Pod时，可以很方便的匹配到同一个数据卷上。

- [使用场景](#使用场景)
- [限制](#限制)
- [组成](#组成)
- [Pod选择器](#Pod选择器)
- [Pod标识](#Pod标识)
- [部署和伸缩的保障](#部署和伸缩的保障)
- [更新策略](#更新策略)
- [下一步……](#下一步)

## 使用场景

当应用有以下需求时，可以考虑使用StatefulSet：

- 稳定、唯一的网络标识。
- 稳定、持久的存储。
- 有序、优雅的部署和伸缩。
- 有序、自动的滚动更新。

在上面列出的内容中，“稳定”就代表在Pod重新调度过程中的持久性。如果你的应用不需要稳定的标识、有序的部署、删除、伸缩，那你应该用无状态副本。这种情况下，[Deployment](Deployment.md)或[ReplicaSet](ReplicaSet.md)对无状态应用可能是更好的选择。

## 限制

- 一个Pod的存储，要么由[持久卷分配器](https://github.com/kubernetes/examples/blob/master/staging/persistent-volume-provisioning/README.md)基于`storage class`进行分配，要么由管理员预分配。
- 对StatefulSet进行删除、缩容时，*不会删除*相关的数据卷。这是为了保证数据的安全性，因为StatefulSet可以被自动的清除，所以数据的安全性相对来说应该是更加重要的。
- 目前，Stateful需要使用[Headless Service]()来为Pod提供网络标识，你需要创建这个Service。
- StatefulSet被删除的时候对于如何停止Pod不提供任何保障。要想有序、优雅的终止里面的Pod，可以在删除之前先将StatefulSet缩容到0，然后再删除。
- 当使用默认的[Pod管理策略](#Pod管理策略)（`OrderedReady`）进行[滚动更新](#滚动更新)时，有可能会进入某种异常状态，此时需要进行[人工干预修复](#强制回滚)。

## 组成

下面的栗子给出了一个StatefulSet的组成部分。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

在这个栗子中：

- 有一个Headless Service，名叫`nginx`，用来控制网络相关的东西。
- StatefulSet名为`web`，包含一个Spec，声明了nginx容器的3个副本，每个运行在唯一的Pod中。
- `volumeClaimTemplates`，使用[持久卷]()提供稳定的存储，由持久卷分配器（PersistentVolume Provisioner）分配。

StatefulSet对象的名字必须是有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)

## Pod选择器

StatefulSet的`.spec.selector`选择器必须要跟`.spec.template.metadata.labels`标签匹配。在1.8版本之前，`.spec.selector`不填的话会有默认值。在1.8之后，不填的话校验不通过。

## Pod标识

StatefulSet的Pod都有一个唯一的标识，包含一个序号、一个稳定的网络身份及稳定的存储。这种身份标识会一直跟着Pod，不论它被调度到哪个节点上。

### 序号

比如StatefulSet有N个副本，每个Pod都会有一个整数序号，从0开始到N-1，在StatefulSet中是唯一的。

### 稳定的网络ID