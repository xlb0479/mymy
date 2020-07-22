# Pod和Service的DNS

本文主要介绍的是k8s提供的DNS服务。

## 介绍

k8s会在集群中调度一套DNS的Pod和Service，通过kubelet，让每个容器使用这套DNS的Service的IP来解析域名。

### 什么样的东西会有域名

集群中的每一个Service（包含DNS服务自身）都会被分配一个域名。默认情况下，一个Pod的DNS搜索列表包含Pod所在的命名空间，以及集群的默认域。我们最好用例子来解释一下：

假设在命名空间`bar`中有一个名为`foo`的Service。那么，同样在`bar`空间内的一个Pod就可以直接用`foo`这个名字找到这个Service了。如果是运行在`quux`空间中的Pod，那就需要用`foo.bar`才能找到这个Service。

下面的内容会详细介绍支持的DNS记录类型以及DNS层次。如果你发现你用了一个本文没有讲过的层次、名字或者查询方式，但是也能用，那就是碰巧，不要心存侥幸，后续可能会发生变化，并且不会通知你。关于最新的规范和定义，参见[k8s中基于DNS的服务发现](https://github.com/kubernetes/dns/blob/master/docs/specification.md)。

## Service

### A/AAAA记录

“普通的”（非headless）Service会得到一个DNS的A或AAAA记录，视Service所用的IP簇而定，对应的名字格式为`my-svc.my-namespace.svc.cluster-domain.example`。这个名字会解析到Service的cluster IP上。

“Headless”（没有cluster IP）的服务，同样也会得到一个DNS的A或AAAA记录，也是根据IP簇而定，名字格式为`my-svc.my-namespace.svc.cluster-domain.example`。但不同于普通的Service，它会解析到由Service匹配的Pod的IP上。客户端会可以得到IP的列表，或者是基于DNS的轮询选择。

### SRV记录

SRV记录是根据命名端口来的，属于普通Service或[Headless Service](Service.md#HeadLess-Service)的一部分。对于每一个命名端口，SRV记录的格式为`_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`。对于普通的Service，它会解析到对应的端口号以及域名：`my-svc.my-namespace.svc.cluster-domain.example`上。对于headless的Service，它会得到多个DNS应答，Service的每个Pod都会对应一个，包含了端口号以及Pod的域名：`auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example`