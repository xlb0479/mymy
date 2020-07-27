# 通过Service来连接应用

## k8s中的容器连接模式

此时你应该已经有一个跑起来的分布式应用了，你可以把它暴露到一个网络中。在我们讨论k8s的网络思想之前，有必要先看看如何用“普通的”方式基于Docker进行网络连接。

默认情况下Docker会使用一个主机私有的网络，所以容器只能跟同一个主机的容器进行通信。Docker容器要想跨节点通信，那就必须再主机自己的IP地址上进行端口分配，然后再转发或代理到容器上。这就意味着，容器调度的时候还要规划端口的调度，或者是进行动态端口分配。

如果要在多个开发者或团队中进行端口的分配规划，这样做的可扩展性是很差的，而且让整件事儿不那么透明，用户要知道这些集群相关的事儿。k8s则认为Pod是可以跟其他Pod通信的，不管它们在哪个主机上。k8s给每个Pod分配它在集群内部的私有IP，因此你不用明确地去创建Pod之间的连接或者是容器端口和主机端口的映射。这就意味着，Pod内部的所有容器可以通过localhost访问对方的端口，集群中的所有Pod在没有NAT的情况下就能看到彼此。本文剩余的部分会详细介绍如果在这种网络模型中运行可靠的服务。

为了验证方便，这里要使用一个Nginx服务器进行演示。

## 将Pod暴露到集群中

这个栗子我们前面见过，我们再拿到这里，这次我们重点关注网络部分。创建一个nginx的Pod，注意它有一个容器端口的定义：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

这样，在集群中任何节点都可以访问到了。查看Pod所在的节点：

```shell script
kubectl apply -f ./run-my-nginx.yaml
kubectl get pods -l run=my-nginx -o wide
```

```text
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
my-nginx-3800858182-jr4a2   1/1       Running   0          13s       10.244.3.4    kubernetes-minion-905m
my-nginx-3800858182-kna2y   1/1       Running   0          13s       10.244.2.5    kubernetes-minion-ljyd
```

查看Pod的IP：

```shell script
kubectl get pods -l run=my-nginx -o yaml | grep podIP
    podIP: 10.244.3.4
    podIP: 10.244.2.5
```

你可以ssh到任意一个节点上curl它们的IP。注意容器*并没有*使用节点的80端口，而且也没有任何特殊的NAT规则把流量路由到Pod。这就意味着你可以在同一个节点上运行多个相同containerPort的Nginx的Pod，而且还可以用IP地址从任意的Pod或节点上访问到它。和Docker类似，端口依旧是支持映射到节点的网络接口上的，但是由于这种网络模型的出现，这种需求基本上没有了。

如果你特好奇，跟个猫似的，你可以去看看[这到底是怎么实现的](../集群管理/集群网络.md)。

## 创建一个Service

此时，我们有了两个Nginx运行在扁平的、集群范围的地址空间中。理论上来说，你可以直接跟Pod进行通信，但如果节点挂了怎么办？Pod跟它同归于尽了，Deployment会创建新的Pod，但是IP不同了。这就是Service要解决的问题。

一个k8s的Service就是一个对一组运行在集群中的Pod的逻辑抽象，这些Pod提供完全相同的功能。创建之后，每个Service会得到一个唯一的IP地址（也叫clusterIP）。这个IP地址是跟这个Service的一生绑定在一起的，只要Service在，它就不会变。Pod可以跟Service进行通信，流量会自动被Service在它的Pod之间进行负载均衡。

你可以用`kubectl expose`给这两个Nginx创建一个Service：

```shell script
kubectl expose deployment/my-nginx
```

```text
service/my-nginx exposed
```

效果跟用`kubectl apply -f`执行下面的yaml文件是一样的：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```