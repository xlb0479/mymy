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

### **iptables**代理模式

这种模式下，kube-proxy监视k8s的control plane来获取Service和Endpoint对象的变更信息。对于每个Service来说，它会添加对应的iptables规则，捕获流向Service的`clusterIP`和`port`的流量，然后将流量重定向到其中一个后端上。对于每个Endpoint对象，它也会生成iptables规则，选择其中一个后端Pod。

该模式默认情况下是随机选择后端的。

使用iptables处理流量，系统负载更小，因为流量是交给Linux的netfilter来处理的，不需要在用户空间和内核空间来回切换。这种方式相对来说更加可靠。

该模式下，如果选择的第一个Pod没有响应，那么链接就会断掉。这就跟用户空间模式不太一样了：在用户空间模式下，会自动选择其他后端Pod进行重试。

可以用Pod的[就绪探针](../业务组件/泡德（Pod）/Pod生命周期.md#容器探针)来判断后端Pod是否OK，这样在该模式下，iptables中就只会保留那些健康的Pod。也就避免了将流量发送到异常Pod上的问题。

![services-iptables-overview.svg](img/services-iptables-overview.svg)

## 多端口

## 自定义IP

## 服务发现

## Headless Service

## 服务发布（ServiceType）

### ExternalName

## 缺陷

## 虚拟IP的实现

## API对象

## 支持的协议

## 下一步……