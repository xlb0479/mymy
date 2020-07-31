# Ingress

**功能状态**：`Kubernetes v1.1 [beta]`

用来管理外部对集群内Service访问的API对象，一般是HTTP。

Ingress可以提供负载均衡、SSL终结以及基于名字的虚拟主机。

## 术语

为了超级清晰，我们定义以下术语：

- 节点（Node）：k8s中的一个机器，属于集群的一部分。
- 集群（Cluster）：由k8s管理，一组运行容器化应用的节点。最常见的k8s部署方式是，所有集群节点都是在内网的。
- 边界路由（Edge router）：为集群执行防火墙策略的路由器。它可能是一个由云服务商管理的网关，或者一个物理硬件设备。
- 集群网络（Cluster network）：一组链接，逻辑的或者是物理的，基于k8s的[网络模型](../集群管理/集群网络.md)，为集群建立通信功能。
- 服务（Service）：一个k8s的[Service](Service.md)，用[标签](../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md)选择器标识了一组Pod。除非特殊提及，我们都假设Service使用的是虚拟IP，只能在集群网络中进行路由。

## Ingress是什么？

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#ingress-v1beta1-networking-k8s-io)暴露了由集群外部到集群内部[Service](Service.md)的HTTP和HTTPS路由。路由规则由Ingress来定义。

```text
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

一个Ingress可以为Service提供外部可以访问的URL、流量负载均衡、SSL/TLS终结，以及可命名的虚拟主机。[Ingress Controller](Ingress%20Controller.md)负责满足Ingress的要求，通常搭配一个负载均衡器，尽管你也可以配置边界路由活其他前端组件来进行流量控制。

Ingress并不是暴露任意的端口或者协议。如果要暴露非HTTP或HTTPS的Service到外部，一般是用[Service.Type=NodePort](Service.md#nodeport)或[Service.Type=LoadBalancer](Service.md#loadbalancer)。

## 前提条件

必须要有一个[Ingress Controller](Ingress%20Controller.md)来满足Ingress。只创建Ingress没有什么卵用。

你可以部署一个基于[ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)的Ingress Controller。可以选择多种[Ingress Controller](Ingress%20Controller.md)。

理想情况下不管是什么样的Ingress Controller都可以满足要求。但实际情况是，不同的Ingress Controller的操作方式也稍有不同。

>**注意**：一定要看过你所选择的Ingress Controller的相关文档，注意其中的警告内容。

## Ingress资源

一个最小化的Ingress栗子：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          serviceName: test
          servicePort: 80
```

和其他资源一样，Ingress也需要`apiVersion`、`kind`以及`metadata`字段（好久没重复这句话了）。Ingress对象的名字必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。对于配置文件的基本常识，自己去看[部署应用](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)、[配置容器](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)、[管理资源](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)。对于具体的Ingress Controller，经常需要在Ingress上面加一些注解，配置一些选项，比如栗子中的[rewrite-target注解](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/rewrite/README.md)。不同的[Ingress Controller](Ingress%20Controller.md)支持的注解也不同。好好看看你用的哪一种，支持哪些注解。

Ingress的[Spec](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)中包含了所有用来配置负载均衡器或代理服务器的信息。最重要的是，它包含了一组规则来匹配到来的请求。Ingress资源只支持对HTTP（S）的规则定义。

### Ingress规则

每个HTTP规则都包含以下信息：

- 一个可选的host。在我们的栗子中，没有host，所以这个规则会用到所有通过IP访问的HTTP请求上。如果定义了host（比如foo.bar.com），那规则就只用在这个host上。
- 一组路径（比如`/testpath`），每个都要关联一个后端，后端信息包括了`serviceName`和`servicePort`。如果负载均衡器要把流量转到这个Service上，那host和path必须同时匹配才行。
- 一个后端包含了Service的名字和端口，见[Service](Service.md)。HTTP（和HTTPS）请求根据host和path的匹配结果，发送到后端中。（咦，这块好像有点不对劲。）

在Ingress Controller中通常会有一个默认后端来处理没有任何匹配的请求。

### 默认后端

一个没有任何规则的Ingress会将所有流量发送到一个单身的默认后端上。这个默认后端通常是[Ingress Controller](Ingress%20Controller.md)的一个配置选项，不在Ingress资源中体现。

如果Ingress对象没有匹配到任何host活path，请求也会被路由到默认后端上。

### Path类型

Ingress中的每个路径都有一个对应的路径类型。目前支持三种路径类型：

- `ImplementationSpecific`（默认）：如果是这种类型，由IngressClass进行匹配。根据具体实现，可以作为单独的一种`pathType`来处理，或者当做跟`Prefix`或`Exact`一样的类型来处理。
- `Exact`：完全匹配URL路径，大小写敏感。
- `Prefix`：基于/分割的路径前缀匹配。大小写敏感，按照每个路径元素依次进行匹配。路径元素就是用/分割后的每个标签。如果每个路径`p`的元素都可以匹配，那么这个请求就可以匹配到这个路径上。

>**注意**：如果路径的最后一个元素是请求路径最后一个元素的子串，则无法形成匹配（比如`/foo/bar`可以匹配`/foo/bar/baz`但是无法匹配`/foo/barbaz`）。

### 多重匹配

一些情况下一个请求可能会匹配到Ingress中的多个路径上。此时要按最长匹配来定。如果进一步发现两个路径匹配长度也相同，则优先匹配Exact类型的路径，其次匹配Prefix类型的路径。

## IngressClass

Ingress由不同的控制器来实现，通常配置也不同。每个Ingress都应该定义一个类型，引用到一个IngressClass资源，其中包含了其他配置信息，包括要实现这种IngressClass的控制器的名字。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com/v1alpha
    kind: IngressParameters
    name: external-lb
```

IngressClass资源包含了一个可选的`parameters`字段。它可以用来引用一些其他的配置信息。

### 废弃的注解们

在1.18版本中出现IngressClass资源以及`ingressClassName`字段之前，Ingress的类型要用注解`kubernetes.io/ingress.class`来声明。这个注解从来没有正式的定义，但是却广泛被各种Ingress Controller支持。

新出的`ingressClassName`属性就是用来替代这个注解的，但它们并不完全等同。以前的注解一般是引用用来实现Ingress的Ingress Controller的名字，而该字段是引用一个IngressClass资源，其中包含了其他Ingress的配置，包括Ingress Controller的名字。

### 默认的IngressClass

可以把某一个IngressClass作为集群的默认选项。将IngressClass的`ingressclass.kubernetes.io/is-default-class`注解值设置为`true`，那么再有新的Ingress，如果没有指定`ingressClassName`那就会用这个默认的IngressClass。

>**警告**：如果集群中被标记为默认的IngressClass数量大于一个，admission controller会拒绝创建没有`ingressClassName`字段的Ingress。你只要保证最多只有一个默认的IngressClass就好了。

## Ingress的类型

### 单服务的Ingress

已经有其他相关概念让你可以暴露一个Service了（见[其他选择](#其他选择)）。你也可以通过为Ingress设置一个没有任何规则的*默认后端*来实现。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```

用`kubectl apply -f`来创建它，查看这个Ingress的状态：

```shell script
kubectl get ingress test-ingress
```

```text
NAME           HOSTS     ADDRESS           PORTS     AGE
test-ingress   *         203.0.113.123   80        59s
```

`203.0.113.123`这个IP是Ingress Controller分配的，用来满足这个Ingress、

>**注意**：Ingress Contrller和负载均衡器需要大概一两分钟才能分配出来一个IP。在此之前，地址那列都是`<pending>`。

### 简单扇出

扇出配置可以从一个单独的IP上将流量路由到多个Service上，根据请求的HTTP URI来定。Ingress可以让负载均衡器的数量降到最低。比如：

```text
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```

它的Ingress就可以是这样：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```

用`kubectl apply -f`创建一下：

```shell script
kubectl describe ingress simple-fanout-example
```

```text
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

IngressController会提供特定实现的负载均衡器来满足这个Ingress，只要Service（`service1`、`service2`）是有效的。完成之后，你就可以在Address字段看到负载均衡器的地址了。

>**注意**：根据你所选择的[Ingress Controller](Ingress%20Controller.md)，可能需要创建一个默认http后端[Service](Service.md)。
>
### 虚拟主机

基于名字的虚拟主机，支持在同一个IP低智商路由多个主机的HTTP请求。

```text
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```

下面这个Ingress，让内部的负载均衡器根据[Host header](https://tools.ietf.org/html/rfc7230#section-5.4)来路由请求。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

如果你创建的Ingress没有定义任何主机信息，那么所有打到Ingress Controller的IP上的流量都可以正常匹配，不需要虚拟主机信息。

比如下面这个Ingress会把`first.bar.com`的请求发给`service1`，把`second.foo.com`的请求发给`service2`，如果不带任何主机信息，直接用IP访问的话（没有请求头），则是打给`service3`。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: second.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
  - http:
      paths:
      - backend:
          serviceName: service3
          servicePort: 80
```

### TLS

可以给Ingress定义一个[Secret](../配置/Secret.md)实现安全通信，其中包含TLS私钥和证书。当前的Ingress只能支持一个TLS端口，就是443，并且都会做TLS终结，不会向后传播。如果Ingress中的TLS配置里包含了不同的主机，那就根据SNI TLS扩展（Ingress Controller要支持SNI）中的主机名形成在同一个端口的多路复用。TLS的Secret中必须包含名为`tls.crt`和`tls.key`的key，分别包含了TLS所需的证书和私钥。比如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

在Ingress中引用这份儿Secret，就可以让Ingress Controller在客户端和负载均衡器之间的通信链路上使用TLS了。必须确保你创建的证书中包含Common Name（CN），也叫作完全限定域名（Fully Qualified Domain Name（FQDN）），比如`sslexample.foo.com`。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - sslexample.foo.com
    secretName: testsecret-tls
  rules:
  - host: sslexample.foo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: service1
          servicePort: 80
```

>**注意**：不同Ingress Controller支持的TLS略有不同。请参考[Nginx](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)、[GCE](https://github.com/kubernetes/ingress-gce/blob/master/README.md#frontend-https)或其他平台特定的Ingress Controller的相关文档，了解你所使用的TLS到底是怎么工作的。

### 负载均衡

一个Ingress Controller自带一套负载均衡策略，应用到所有的Ingress上，比如负载均衡算法、后端权重计算等等。更高级的负载均衡概念（比如会话保持、动态权重）并没有暴露到Ingress中。你可以在Service使用的负载均衡器上搞这些小伎俩。

值得注意的是，虽然Ingress中没有直接体现健康检查的概念，但是像[就绪探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)这种k8s已有的概念可以让你实现同样的效果。请你去看控制器的相关文档，了解它们是如何处理健康检查的（[Nginx](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)、[GCE](https://github.com/kubernetes/ingress-gce/blob/master/README.md#health-checks)）。

## 更新Ingress

如果要更新已有的Ingress，添加一个新的Host，可以直接编辑资源进行更新：

```shell script
kubectl describe ingress test
```

```text
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     35s                loadbalancer-controller  default/test
```

```shell script
kubectl edit ingress test
```

此时会弹出一个编辑器，包含YAML格式的当前配置。直接修改，添加新的Host：

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /foo
  - host: bar.baz.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
        path: /foo
..
```

当你改完保存之后，kubectl会自动更新API server中的资源信息，而后进一步触发Ingress Controller去重新配置负载均衡器。

看一下：

```shell script
kubectl describe ingress test
```

```text
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
  bar.baz.com
               /foo   service2:80 (10.8.0.91:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     45s                loadbalancer-controller  default/test
```

除了这种方式，还可以修改已有的Ingress YAML文件，执行`kubectl replace -f`，也是一样的效果。

## 跨可用区的异常

如何调整多个异常域之间的流量，每个云服务商的策略也不尽相同。可以查看相关的[Ingress Controller](Ingress%20Controller.md)文档了解详细情况。还有[Federation文档](https://github.com/kubernetes-sigs/kubefed)，其中包含了在Federated集群中部署Ingress的相关细节。

## 展望未来

关注[SIG Network](https://github.com/kubernetes/community/tree/master/sig-network)，了解Ingress及相关资源的最新研究进展。还有[Ingress Repository](https://github.com/kubernetes/ingress-nginx/tree/master)，包含各种Ingress Controller的进展情况。

## 其他选择

有多重方式暴露Service，不比拘泥于Ingress：

- [Service.Type=LoadBalancer](Service.md#loadbalancer)
- [Service.Type=NodePort](Service.md#nodeport)

## 下一步……

- 学习[Ingress API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#ingress-v1beta1-networking-k8s-io)
- 学习[Ingress Controller](Ingress%20Controller.md)
- [在Minikube中创建一个Ingress以及NGINX Controller](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)