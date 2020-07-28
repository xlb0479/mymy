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

这个定义会创建一个Service，指向所有带`run: my-nginx`标签的Pod的80端口，暴露到一个抽象的Service端口上（`targetPort`：容器接收流量的端口，`port`：抽象Service端口，可以是任意端口，就是为了让其他Pod来访问这个Service）。关于Service支持的属性可以去看[Service](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#service-v1-core) API对象。检查一下我们创建的Service：

```shell script
kubectl get svc my-nginx
```

```text
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.0.162.149   <none>        80/TCP    21s
```

前文书我们讲过，一个Service后面是一组Pod。这些Pod是通过`endpoint`暴露的。Service的选择器会持续进行计算，将结果POST到同样名为`my-nginx`的Endpoint对象上。当一个Pod挂了，会自动从endpoint中移除。来看一下endpoint，注意它们的IP跟第一步创建的Pod的IP是一样的：

```shell script
kubectl describe svc my-nginx
```

```text
Name:                my-nginx
Namespace:           default
Labels:              run=my-nginx
Annotations:         <none>
Selector:            run=my-nginx
Type:                ClusterIP
IP:                  10.0.162.149
Port:                <unset> 80/TCP
Endpoints:           10.244.2.5:80,10.244.3.4:80
Session Affinity:    None
Events:              <none>
```

```shell script
kubectl get ep my-nginx
```

```text
NAME       ENDPOINTS                     AGE
my-nginx   10.244.2.5:80,10.244.3.4:80   1m
```

此时，你就可以用curl通过`<CLUSTER-IP>:<PORT>`看一下这个Nginx的Service了，集群中的任意一个节点都可以。注意这个Service的IP完全是虚拟的，绝对不会接触底层物理网络。如果你特好奇，那你可以去看一下[服务代理](Service.md)。

## 访问Service

k8s支持两种主要的模式来找到一个Service——环境变量和DNS。前者开箱即用，后者需要[CoreDNS插件](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/coredns)。

>**注意**：如果环境变量不足以满足需求（变量冲突、变量过多、只用DNS等）你可以在[Pod定义](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#pod-v1-core)时将`enableServiceLinks`设置为`false`来关闭这种模式。

### 环境变量

当一个Pod运行在节点上，kubelet会将每个活动的Service相关的环境变量注入到Pod中。这就导致了一个顺序问题。想知道为什么？检查一下你的Nginx的Pod的环境变量（你的实际Pod名字可能会不一样）：

```shell script
kubectl exec my-nginx-3800858182-jr4a2 -- printenv | grep SERVICE
```

```text
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```

注意，这里并没有你刚才创建的Service。这是因为Pod是在Service之前创建的。另一个问题是，调度器可能会把两个Pod调度到同一台机器上，如果这个机器挂了，那你的Service整个就完犊子了。我们可以把这两个Pod都杀掉，等Deployment进行重建。这次，Service就在Pod*之前*了。这就可以为Service提供在调度器层面的Pod分布控制（每个节点的容量都保持一致），也就会有正确的环境变量了：

```shell script
kubectl scale deployment my-nginx --replicas=0; kubectl scale deployment my-nginx --replicas=2;

kubectl get pods -l run=my-nginx -o wide
```

```text
NAME                        READY     STATUS    RESTARTS   AGE     IP            NODE
my-nginx-3800858182-e9ihh   1/1       Running   0          5s      10.244.2.7    kubernetes-minion-ljyd
my-nginx-3800858182-j4rm4   1/1       Running   0          5s      10.244.3.8    kubernetes-minion-905m
```

你会发现Pod的名字变了，因为它们被杀掉重建了。

```shell script
kubectl exec my-nginx-3800858182-e9ihh -- printenv | grep SERVICE
```

```text
KUBERNETES_SERVICE_PORT=443
MY_NGINX_SERVICE_HOST=10.0.162.149
KUBERNETES_SERVICE_HOST=10.0.0.1
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_PORT_HTTPS=443
```

### DNS

k8s提供了一个集群DNS插件Service，自动为其他Service生成DNS记录。我们来检查一下当前的状况：

```shell script
kubectl get services kube-dns --namespace=kube-system
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   8m
```

下面的内容假设你已经有一个稳定的Service了（my-nginx），并且DNS服务已经给它分配了一个记录。我们这里用的是CoreDNS集群插件（应用名为`kube-dns`），因此，你可以在集群中的任意Pod中使用标准方法（比如`gethostbyname()`）来访问到这个Service。如果CoreDNS没有运行，可以参考[CoreDNS README](https://github.com/coredns/deployment/tree/master/kubernetes)或[安装CoreDNS](https://kubernetes.io/docs/tasks/administer-cluster/coredns/#installing-coredns)来开启它。我们来运行一个curl应用进行测试：

```shell script
kubectl run curl --image=radial/busyboxplus:curl -i --tty
```

```text
Waiting for pod default/curl-131556218-9fnch to be running, status is Pending, pod ready: false
Hit enter for command prompt
```

好了，我们执行一下`nslookup my-nginx`：

```text
[ root@curl-131556218-9fnch:/ ]$ nslookup my-nginx
Server:    10.0.0.10
Address 1: 10.0.0.10

Name:      my-nginx
Address 1: 10.0.162.149
```