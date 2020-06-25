# Service

定义：将一个运行在一组[Pod](../业务组件/泡德（Pod）/概要.md)上的应用抽象地暴露成一个网络服务。

有了k8s就不用再为了用一个你不熟悉的服务发现机制而修改你的程序了。k8s为每个Pod提供了它们自己的IP地址，可以为一组Pod提供一个DNS名称，并且可以在它们之间实现负载均衡。

- [动机](#动机)
- [Service资源](#Service资源)
- [定义Service](#定义Service)
- [虚拟IP和服务代理](#虚拟IP和服务代理)
- [多端口](#多端口)
- [自定义IP](#自定义IP)
- [服务发现](#服务发现)
- [Headless Service](#HeadLess-Service)
- [服务发布（ServiceType）](#服务发布ServiceType)
- [缺陷](#缺陷)
- [虚拟IP的实现](#虚拟IP的实现)
- [API对象](#API对象)
- [支持的协议](#支持的协议)
- [下一步……](#下一步)

## 动机

[Pod](../业务组件/泡德（Pod）/概要.md)是屌丝。它们由生入死，不存在复活一说。如果你用了[Deployment](../业务组件/控制器/Deployment.md)，它倒是可以动态的创建和删除Pod。

每个Pod都有自己的IP，但是在Deployment下，当前运行的一组Pod，过了一会儿有可能就变成另一个样了。

这就导致一个问题：如果某些Pod（称为“后端”）是为集群中其它Pod（称为“前端”）提供一些功能支持，那么前端的Pod该怎么知道并持续跟踪它们要连接的后端Pod的IP地址呢，如何才能把后端用起来呢？

那么我就让你见识一下什么叫*Service*。

## Service资源

在k8s中，Service是一种抽象（一说抽象是不是就感觉有点抽象），它定义了一个包含Pod的逻辑集合，以及一个访问这些Pod的策略（有时我们把这种模式称为微服务）。Service通常是使用[选择器](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)来确定它所指向的Pod集合（下面[有个地方](#不带选择器的Service)会说到某个时候可能你的Service*不需要*选择器）。

比如一个无状态的用来做镜像处理的后端程序，运行了3个副本。这些副本都是可以互相替代的——前端并不关系它们用的是哪个后端。而真正组成后端的那些Pod可能会发生改变，前端并不需要知道这些屁事儿，它们也不需要自己去跟踪它们所使用的后端。

Service抽象就是实现了这种解耦。

### 云原生服务发现

如果你的程序中调用了k8s的API来做服务发现，你可以从[apiserver]查询Endpoint，当Service中的Pod发生变化时它们也会跟着变化。

对于非原生程序，k8s可以在你的程序和后端Pod之间提供一个网络端口或者负载均衡器。

## 定义Service

Service就是一个REST对象，跟Pod一样。跟所有REST对象一样，可以把一个Service的定义`POST`到apiserver，从而创建一个实例。Service对象的名字必须是一个有效的[DNS标签名](../概要/Kubernetes对象/对象的名字和ID.md#DNS标签名)。

比如你有一组Pod，每个Pod监听TCP端口9376，并且带着一个`app=MyApp`标签：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

这个定义就会创建一个名为“my-service”的Service对象，它的目标端口是所有带`app=MyApp`标签Pod的9376端口。

k8s会为这个Service也提供一个IP（有的时候称为“集群IP”），用于Service代理（见下面的[虚拟IP和服务代理](虚拟IP和服务代理)）。

控制器会根据Service的选择器持续地观察是否有匹配的Pod，并更新同样名为“my-service”的Endpoint对象。

>**注意**：一个Service可以把*任意*入口`port`指向`targetPort`。默认情况下，为了简单，`targetPort`等于`port`。

Pod中的端口定义都是有名字的，你可以在Service的`targetPort`属性中引用这些名字。即便是在Service下混合了多种Pod，都配了同一个名字，这种方法也同样能用，它们可以基于同样的协议但是使用不同的端口（把协议名作为端口名就可以了）。这为Service的部署和后续演变提供了非常大的便利。举例来说，对于一个后端软件，下个版本你可以修改Pod暴露的端口号，但是不需要修改客户端的软件。

Service默认的协议是TCP；也可以使用其他[支持的协议](#支持的协议)。

很多Service可能需要暴露不止一个端口，k8s支持在Service对象中定义多个端口。每个端口定义中的`protocol`可以不同。

### 不带选择器的Service

Service一般都是用来对Pod的访问方式进行抽象，但也可以对一些其他的后端进行抽象。比如：

- 你希望在生产环境中使用外部的数据库，但是测试环境中使用自己的数据库。
- 你想让Service指向另一个位于不同[命名空间（Namespace）](../概要/Kubernetes对象/命名空间.md)或不同集群的Service。
- 你想把一些服务迁移到k8s中，在验证阶段，你希望只在k8s中部署其中的一部分后端服务。

在上面提到的场景中，你都可以定义一个*不带*选择器的Service。比如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

因为这个Service没有选择器，所以*不会*自动创建对应的Endpoint对象。你需要手动添加一个Endpoint对象，将Service映射到它所指向的网络地址和端口：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

Endpoint对象的名字必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

>**注意**：Endpoint的IP*绝对不能*是：环回接口（127.0.0.0/8（IPv4），::1/128（IPv6）），链路本地地址（169.254.0.0/16和224.0.0.0/24（IPv4），fe80::/64（IPv6））。同样，也不能是其他Service的集群IP，这是因为[kube-proxy]()不支持以虚拟IP为目标地址。

访问无选择器的Service没有什么不一样。在上面的栗子中，流量会被路由到YAML中定义的端点上：`192.0.2.42:9376`（TCP）。

如果Service的类型为ExternalName，这是一种无选择器Service特例，它用的是DNS域名。详见本文中的[相关介绍](#ExternalName)。

### EndpointSlice

**功能状态**：`Kubernetes v1.17 [beta]`

EndpointSlice也是一种API资源，它可以提供比Endpoint更好的可伸缩性。尽管概念上可能比较类似，EndpointSlice可以在多个资源上分布网络端点。默认情况下EndpointSlice的端点数量达到100就被认为“满了”，会创建其它的EndpointSlice来存储额外的端点信息。

EndpointSlice还提供了其它的属性和功能，详见[EndpointSlice](EndpointSlice.md)。

### 应用协议

**功能状态**：`Kubernetes v1.18 [alpha]`

可以在每个Service端口上设置AppProtocol字段，用于声明所使用的应用协议。

由于是个alpha阶段的功能，默认这个字段是不启用的。你要想用，需要开启`ServiceAppProtocol`[特性门]()。

## 虚拟IP和服务代理

k8s集群中的每个节点都跑着一个`kube-proxy`。`kube-proxy`负责提供一种虚拟IP实现，除了类型为[`ExternalName`](#ExternalName)的Service外，其他Service都需要用到。

### 为什么不基于DNS做轮询

时不时就会提到这个问题，为什么k8s要依赖代理将入口流量转发到后端。其他方法不香吗？比如配置多个A记录的DNS记录（或者是IPv6的AAAA），然后基于轮询做域名解析？

之所以让Service使用代理，有以下原因：

- 在历史的长河中，DNS在实现的时候经常不理会TTL值，在记录本该过期的时候依然缓存着它们的查询结果。
- 有些应用做了一次DNS查询缓存一辈子。
- 即便应用和程序库做了适当的重新解析机制，如果TTL过低或直接是零值，会对DNS造成很高的负载，很难管理。

### 用户空间代理模式

这种模式下，kube-proxy通过监视k8s的master来获取Service和Endpoint对象的变化。对于每个Service它会在本地节点上打开一个端口（随机选择）。所有连接到这个“代理端口”上的链接，都会被代理到Service的其中一个后端Pod上（基于Endpoint信息）。kube-proxy会基于Service的`SessionAffinity`来判断该使用哪个Pod。

最后要说的是，这种模式会在iptables中生成各种规则，捕获那些指向Service的`clusterIP`（虚拟IP）和`port`的流量。这些规则会将流量定向到代理端口，然后进一步代理到后端Pod上。

用户空间代理模式在默认情况下是根据轮询算法来选择后端Pod的。

![services-userspace-overview.svg](img/services-userspace-overview.svg)

### `iptables`代理模式

这种模式下，kube-proxy监视k8s的control plane来获取Service和Endpoint对象的变更信息。对于每个Service来说，它会添加对应的iptables规则，捕获流向Service的`clusterIP`和`port`的流量，然后将流量重定向到其中一个后端上。对于每个Endpoint对象，它也会生成iptables规则，选择其中一个后端Pod。

该模式默认情况下是随机选择后端的。

使用iptables处理流量，系统负载更小，因为流量是交给Linux的netfilter来处理的，不需要在用户空间和内核空间来回切换。这种方式相对来说更加可靠。

该模式下，如果选择的第一个Pod没有响应，那么链接就会断掉。这就跟用户空间模式不太一样了：在用户空间模式下，会自动选择其他后端Pod进行重试。

可以用Pod的[就绪探针](../业务组件/泡德（Pod）/Pod生命周期.md#容器探针)来判断后端Pod是否OK，这样在该模式下，iptables中就只会保留那些健康的Pod。也就避免了将流量发送到异常Pod上的问题。

![services-iptables-overview.svg](img/services-iptables-overview.svg)

### IPVS代理模式

**功能状态**：`Kubernetes v1.11 [stable]`

在`ipvs`模式下，kube-proxy监视着k8s的Service和Endpoint，根据变化情况去调用`netlink`接口来创建IPVS规则，并周期性地将这些规则和Service以及Endpoint进行同步。这种控制循环可以确保IPVS的状态能够匹配实际状态。当访问一个Service时，IPVS就会把流量转发到其中一个后端Pod上。

IPVS代理模式是基于netfilter的钩子函数，这跟iptables模式有点类似，但IPVS在内核空间工作的同时，还使用了哈希表作为底层数据结构。这就意味着IPVS模式在流量转发上比iptables模式的延迟更低，同步代理规则时的性能更好。相较于其他代理模式，IPVS可支持的网络吞吐量也更高。

对于后端Pod的流量均衡方式，IPVS还提供了很多选项：

- `rr`：轮询
- `lc`：最小连接数（最少的打开连接的数量）
- `dh`：目标地址哈希
- `sh`：源地址哈希
- `sed`：最小期望延迟
- `nq`：从不排队

>**注意**：要想让kube-proxy运行在IPVS模式下，必须在启动它之前先开启节点上的IPVS。kube-proxy使用IPVS模式启动时会检查IPVS内核模块是否可用。如果没有发现该内核模块，kube-proxy会退回到iptables的代理模式。

![services-ipvs-overview.svg](img/services-ipvs-overview.svg)

这种模式下，Service的IP:Port上的流量会被代理到合适的后端Pod上，对于客户端来说是完全透明的。

如果你想让某个客户端的连接始终被转发到同一个Pod上，可以使用基于客户端IP的session粘性，将`service.spec.sessionAffinity`设置为“ClientIP”（默认是“None”）。还可以设置session粘性的超时时间`service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`。（默认是10800，也就是3个小时，所以请不要一直盯着屏幕3个小时不换台）。

## 多端口

有些服务需要暴露多个端口。k8s允许在Service对象中配置多个端口。当你使用了多个端口，必须给所有端口起一个不冲突的名字。比如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

>**注意**：根据k8s的[命名规范](../概要/Kubernetes对象/对象的名字和ID.md)，端口的名字只能包含小写字母数字和`-`减号。必须以字母数字开头结尾。比如`123-abc`和`web`就没问题，但不能是`123_abc`或`-web`。

## 自定义IP

创建Service的时候也可以自定义Service的集群IP。也就是设置`.spec.clusterIP`字段。比如你想重用一个已有的DNS记录，或者说某个遗留系统配的是固定IP，很难改变的那种。

设置的IP地址必须是有效的IPv4或IPv6地址，必须包含在apiserver的`service-cluster-ip-range`范围内。如果你设置的clusterIP无效，apiserver会返回HTTP状态码422，表示创建过程中出现问题。

## 服务发现

对于如何发现一个Service，k8s主要由两种模式——环境变量和DNS。

### 环境变量

当一个Pod运行起来之后，kubelet会将每一个活动Service的相关环境变量加到Pod容器中。它既可以兼容[Docker连接](https://docs.docker.com/network/links/)变量（见[makeLinkVariables](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/envvars/envvars.go#L72)），也支持简单的`{SVCNAME}_SERVICE_HOST`和`{SVCNAME}_SERVICE_PORT`，其中Service名字是全大写，减号转成下划线。

比如，`“redis-master”`这个服务暴露了TCP端口6379，分配的集群IP是10.0.0.11，就会生成以下环境变量：

```properties
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

>**注意**：当你的Pod需要访问一个Service时，如果你想用环境变量的方式将Service的端口和集群IP暴露给Pod，那你必须在Pod创建*之前*先创建好Service。
>否则Pod中就不会注入Service的环境变量。如果你只用DNS做Service的发现，那你就不用操心这种顺序问题了。

### DNS

你，可以（或者说总是应该）使用[add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/)的方式为k8s集群设置一个DNS服务。

集群范围的DNS，比如一个CoreDNS吧，可以监视新建Service相关的API，为每个Service创建一组DNS记录。如果你在集群中启用了DNS，所有Pod就可以自动使用DNS名来找到Service。

比如在命名空间`“my-ns”`中有一个名为`“my-service”`的Service，control plane和DNS服务就会协同工作，创建一个DNS记录`“my-service.my-ns”`。在`“my-ns”`中的Pod就可以直接用`my-service`这个名字找到Service（用`“my-service.my-ns”`也可以）。

其他命名空间中的Pod就必须要用限定名`my-service.my-ns`。不论用哪种名字，都会解析到Service的集群IP上。

对于命名端口，k8s还可以为Service提供SRV记录值。比如`“my-service.my-ns”`这个Service有一个名为`“http”`的端口，协议设置为`TCP`，那你就可以查询`_http._tcp.my-service.my-ns`的SRV值，从而发现它的`“http”`端口以及IP地址。

要想访问`ExternalName`Service就必须使用k8s的DNS服务。关于这部分的内容可以去看[Pod和Service的DNS](Pod和Service的DNS.md)。

## Headless Service

有的时候可能你不需要Service提供负载均衡和IP。此时就可以将集群IP（`.spec.clusterIP`）设置为`“None”`，创建所谓的“无头”Service。

对于无头的`Service`，不会为其分配集群IP，kube-proxy也不会打理这种Service，平台层面不会为其提供任何负载均衡或代理机制。DNS的自动配置依赖于Service是否定义了选择器：

### 有选择器

对于有选择器的无头Service，Endpoint控制器会创建`Endpoint`记录，将DNS配置成直接返回`Service`后端的`Pod`地址。

### 无选择器

对于无选择器的无头Service，Endpoint控制器不会创建`Endpoint`记录。但DNS系统会查询并配置以下：

- [ExternalName](#ExternalName)Service的CNAME记录。
- 对于其他类型的Service，如果有`Endpoint`和Service共享同一个名字，也会创建一条记录。（嗯？这是什么意思？）

## 服务发布（ServiceType）

对于应用中的某些部分（比如前端应用）你可能需要将Service暴露到外部地址上，可以从集群外访问。

k8s的`ServiceType`可以允许你定义Service的类型。默认类型是`ClusterIP`。

`Type`的值及其作用：

- `ClusterIP`：将Service暴露到集群内部的IP上。这种类型的Service只能在集群内部访问。这也是默认的`ServiceType`。
- [`NodePort`](#NodePort)：将服务暴露到每个节点IP的固定端口上（`NodePort`）。系统会为这个`NodePort`Service自动创建一个`ClusterIP`Service，前者会路由到后者身上。你可以使用`<NodeIP>:<NodePort>`的方式从集群外部访问这个`NodePort`Service。
- [`LoadBalancer`](#LoadBalancer)：通过云服务商提供的负载均衡器将Service暴露到外界。会自动创建`NodePort`和`ClusterIP`Service，将外部的负载均衡器路由到它们身上。
- [`ExternalName`](#ExternalName)：通过`CNAME`记录将Service映射到`externalName`字段指定的名字上（比如`foo.bar.example.com`）。这种类型不会有任何代理机制。

>**注意**：要想使用`ExternalName`类型，必须使用1.7版本的kube-dns或者0.0.8及以后版本的CoreDNS。

你也可以使用[Ingress](Ingress.md)来暴露Service。Ingress并不是一种Service，但它可以用作集群的入口点。它可以让你的某一种资源的路由规则稳定性更好，因为它可以将多个服务暴露到相同的IP地址上。

### NodePort

如果将`type`字段设置为`NodePort`，k8s的control plane会分配一个由`--service-node-port-range`选项（默认30000~32767）指定范围内的端口。每个节点会将这个端口（每个节点上都是同一个端口）代理到你的服务上。可以在Service的`.spec.ports[*].nodePort`字段中看到这个端口的具体值。

如果想用指定IP来代理这个端口，可以用kube-proxy的`--nodeport-addresses`选项来设置指定的IP段；这种奇淫巧技是从1.10版本开始有的。这个选项支持逗号间隔的IP段参数（比如10.0.0.0/8，192.0.2.0/25），让kube-proxy认为这些IP范围是节点的本地地址。

举个栗子，启动kube-proxy时设置`--nodeport-addresses=127.0.0.0/8`，那么kube-proxy就只会为NodePort类型的Service使用环回接口。`--nodeport-addresses`的默认值时一个空列表。这就让kube-proxy把所有的可用网络接口都用于NodePort。（这样也可以跟以前的k8s版本兼容）。

如果你想指定一个端口，那就给`nodePort`字段设置一个值。control plane要么为你分配好你需要的端口，要么就在API调用时报错了。这就意味着你需要自己去提前考虑好可能出现的端口冲突的问题。同时你必须使用有效的端口，必须在指定的端口范围内。

使用NodePort可以让你自己去设计负载均衡方案，可以配置那些k8s还无法支持的环境，或者直接暴露某几个节点的IP出来。

注意这种服务可以使用`<NodeIP>:spec.ports[*].nodePort`和`.spec.clusterIP:spec.ports[*].port`两种方式来访问。（如果设置了kube-proxy的`--nodeport-addresses`，那NodeIP就是根据规则过滤后的IP了。）

例如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

### LoadBalancer

对于可以支持外部负载均衡器的云服务商来说，可以将`type`字段设置为`LoadBalancer`，为Service分配负载均衡器。负载均衡器的实际创建时异步发生的，分配的负载均衡器的信息在Service的`.status.loadBalancer`字段中。比如这样：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

外部负载均衡器的流量直接导向后端的Pod上。由云服务商决定具体的负载均衡算法。

对于这种类型的Service，如果定义了多个端口，所有端口必须使用相同的协议字段，且协议必须是`TCP`、`UDP`或`SCTP`之一。

有些云服务商允许设置`loadBalancerIP`。这种情况下会根据用户指定的`loadBalancerIP`来创建负载均衡器。如果没有指定该字段，会使用临时IP来创建负载均衡器。如果你设置了这个字段但是云服务商却不支持，该字段直接被忽略。

>**注意**：如果你用了SCTP协议，重点关注下面关于LoadBalancer类型的[一些问题](#LoadBalancer类型的Service)。

>**注意**：这里原文时关于**Azure**的一些特殊问题，我就不写了，反正老子不用。

#### 内部负载均衡器

在一些混合环境中，可能有那种对同一个网络中的内部服务流量进行路由的场景。

在水平分割的DNS环境中，需要两个Service分别将外部和内部的流量路由到Endpoint上。

可以在Service上增加以下注解来实现这种功能。具体注解依赖于你所使用的云服务商。这里我就列出几个常用的，其他的就不闹了。

- 阿里云

```yaml
[...]
metadata:
  annotations:  
    service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
[...]
```

- 腾讯云

```yaml
[...]
metadata:
  annotations:  
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxx
[...]
```

- 百度云

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/cce-load-balancer-internal-vpc: "true"
[...]
```

- AWS

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-internal: "true"
[...]
```

#### 下面都是关于云服务商的一些特定问题的解决方案，我就不在这里写了，我的翻译尽量做到只关注k8s本身的东西

### ExternalName

这种类型的Service可以把Service映射到一个DNS域名上，而非普通的`my-service`或`cassandra`选择器。这种Service需要设置`spec.externalName`。

举个栗子，我们将`prod`命名空间下的`my-service`这个Service映射到`my.database.example.com`上：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

>**注意**：ExternalName可以接受一个IPv4地址字符串，但是是作为一个数字组成的DNS域名，而不是IP地址。ExternalName不会经由CoreDNS或ingress-nginx解析，因为这个东西本来就是要作为一个完整的DNS域名。如果你想硬编码一个IP地址，可以去用[无头Service](#Headless-Service)。

当查询主机`my-service.prod.svc.cluster.local`时，集群DNS服务会返回一个值为`my.database.example.com`的`CNAME`记录。访问`my-service`的时候看上去跟其他Service没什么两样，但是在DNS层面上有着巨大的差异，因为这里发生的是重定向，而非代理或者转发。后期你可以考虑把数据库挪到集群里，启动Pod，添加合适的选择器或Endpoint，然后修改Service的`type`。

>**警告**：
>对于一些常见的协议，使用ExternalName的时候可能会遇到问题，包括HTTP和HTTPS。如果你使用ExternalName，集群中客户端使用的主机名并不是ExternalName引用的那个名字。
>
>对于需要主机名的协议，这种差异可能会导致错误或无法预料的响应。HTTP请求需要一个`Host:`头，最初的服务端是无法识别的；TLS服务端无法提供跟客户端所连接的主机名匹配的证书。

>**注意**：诸部份的内容要感谢[Alen Komljen](https://akomljen.com/)所写的博文[Kubernetes Tips - Part 1](https://akomljen.com/kubernetes-tips-part-1/)。

### External IP

如果由外部IP地址路由到了集群中的一个或多个节点上，Service可以暴露到这些`externalIPs`上。从这些外部IP（作为目的IP）+Service端口进入集群的流量，会被路由到Service的其中一个Endpoint上。`externalIPs`不是由k8s管理的，需要集群管理员自己去负责。

在具体定义方面，无论哪种Service类型都可以设置`externalIPs`。在下面的栗子中，客户端可以用“`80.11.12.10:80`”（`externalIP:port`）来访问“`my-service`”。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

## 缺陷

对于用户空间模式的VIP代理，尽量用在中小规模的集群，但不要用在那种上千个Service的超大集群中。具体为什么可以去看[这里](https://github.com/kubernetes/kubernetes/issues/1107)。

使用用户空间代理访问Service会丢掉数据包中的源IP。这可能会影响某些网络过滤（防火墙）功能。iptables的代理模式不会丢掉集群内访问时的源IP，但是依然会影响那些从负载均衡器或NodePort连进来的客户端。

`Type`字段被设计成了嵌套型的功能——每层加到前一层上。并不是所有云服务商都有这种严格要求（比如GCE不需要`LoadBalancer`必须有`NodePort`，但是AWS就要），但目前的API都是需要的。

## 虚拟IP的实现

对于那些只是想使用Service的人来说，上面讲的东西已经足够了。但是有一些底层的东西也是值得看上一看的。

### 冲突避免



## API对象

## 支持的协议

## 下一步……