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

SRV记录是根据命名端口来的，属于普通Service或[Headless Service](Service.md#HeadLess-Service)的一部分。对于每一个命名端口，SRV记录的格式为`_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`。对于普通的Service，它会解析到对应的端口号以及域名：`my-svc.my-namespace.svc.cluster-domain.example`上。对于headless的Service，它会得到多个DNS应答，Service的每个Pod都会对应一个，包含了端口号以及Pod的域名：`auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example`。

## Pod

### A/AAAA记录

任何基于Deployment活DaemonSet创建的Pod都可以使用如下格式的DNS解析：

`pod-ip-address.deployment-name.my-namespace.svc.cluster-domain.example`。

### Pod的hostname与子域名字段

当前版本中的Pod在创建之后得到的hostname就是它的`metadata.name`属性的值。

Pod定义时还支持一个可选的`hostname`字段，可以用来指定Pod的主机名。设置了之后，它的优先级就会高于Pod的名字，用作主机名。比如一个Pod的`hostname`设置为“`my-host`”，那它的主机名就是“`my-host`”。

Pod定义时还有另一个可选字段，`subdomain`，可以用来定义子域名。比如一个Pod的`hostname`设置成了“`foo`”，`subdomain`设置成了“`bar`”，命名空间为“`my-namespace`”，那它的完全限定名（FQDN）就是“`foo.bar.my-namespace.svc.cluster-domain.example`”。

栗子：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # Actually, no port is needed.
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```

看上面这个栗子，如果一个headless的Service，跟同一个命名空间下的pod的子域名一致，集群的DNS可以返回Pod的完全主机名对应的A或AAAA记录。比如在这个例子中，其中一个Pod的主机名为“`busybox-1`”，子域名为“`default-subdomain`”，此时，同一个命名空间下的一个headless service同样名为“`default-subdomain`”，那么Pod就会得到它的FQDN“`busybox-1.default-subdomain.my-namespace.svc.cluster-domain.example`”。DNS基于这个名字提供A或AAAA记录，指向Pod的IP地址。“`busybox1`”和“`busybox2`”拥有不同的A或AAAA记录。

Endpoint对象可以为任意端点地址及其IP设置`hostname`。

>**注意**：A或AAAA记录并不是为了作为Pod的名字而创建的，要想创建记录，Pod必须要有`hostname`。一个没有`hostname`但有`subdomain`的Pod只会基于它的headless service创建A或AAAA记录（`default-subdomain.my-namespace.svc.cluster-domain.example`），指向Pod的IP地址。而且，Pod要想真的能够生成记录值，必须要进入就绪（ready）状态，除非在Service上声明了`publishNotReadyAddresses=True`。
>
### DNS策略

可以在Pod这一层级上设置DNS策略。当前k8s支持一下Pod级别的DNS策略。这些策略定义在Pod的`dnsPolicy`字段中。

- “`Default`”：Pod从所在的节点上继承域名解析配置。参见[有关文章](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#inheriting-dns-from-the-node)。
- “`ClusterFirst`”：任何跟集群域名后缀不匹配的DNS查询，比如“`www.kubernetes.io`”，会被转发到所在节点配置的上游DNS服务器。集群管理员可能还配置了额外的stub-domain和上游DNS服务器。相关场景可以参见[有关文章](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#effects-on-pods)。
- “`ClusterFirstWithHostNet`”：对于运行在hostNetwork中的Pod，必须明确将它的DNS策略设置为“`ClusterFirstWithHostNet`”。
- “`None`”：允许Pod忽略k8s的DNS设置。所有DNS设置都来自于Pod定义中的`dnsConfig`字段。参加下面的[DNS配置](#DNS配置)。

>**注意**：“Default”并不是默认的DNS策略（蛤？？？）。如果没有声明`dnsPolicy`，默认是“ClusterFirst”。（蛤？？？）

下面的Pod的DNS策略设置为“ClusterFirstWithHostNet”，因为它把`hostNetwork`设置为`true`了。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

### DNS配置

Pod的DNS配置可以让用户更强势地控制一个Pod的DNS设置。

`dnsConfig`字段是可选的，可以跟任意`dnsPolicy`搭配使用。但是，如果Pod的`dnsPolicy`设置为“`None`”，那么`dnsConfig`字段就必须要要设置。

下面是`dnsConfig`字段支持的各种属性：

- `nameservers`：用作Pod的DNS服务器的IP列表。最多3个IP。当Pod的`dnsPolicy`为“`None`”时，这里至少要包含一个IP，否则就可有可无。这里列出的IP会跟基于DNS策略产生的DNS服务器列表进行合并，去除重复的IP。
- `searches`：DNS搜索域列表，用于在Pod中进行主机名查询。这个字段是可选的。如果填了，这个列表会并入到基于DNS策略生成的搜索域列表中。重复的域名会被去除。允许最多6个搜索域。
- `options`：可选字段。这是一个对象列表，每个对象有一个`name`属性（必填）和一个`value`属性（选填）。这里的内容同样会并入基于DNS策略产生的options中，重复的条目会被去除。

下面是一个自定义DNS设置的Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

如果按照这样创建一个Pod，`test`容器的`/etc/resolv.conf`文件内容如下：

```yaml
nameserver 1.2.3.4
search ns1.svc.cluster-domain.example my.dns.search.suffix
options ndots:2 edns0
```

对于IPv6，应该像下面这样：

```shell script
kubectl exec -it dns-example -- cat /etc/resolv.conf
```

输出：

```text
nameserver fd00:79:30::a
search default.svc.cluster-domain.example svc.cluster-domain.example cluster-domain.example
options ndots:5
```

#### 支持的版本

Pod的DNS设置及DNS策略“`None`”支持的版本如下。

k8s版本|功能版本
-|-
1.14|Stable
1.10|Beta（默认打开）
1.9|Alpha

## 下一步……

[如何管理DNS](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)。