# IPv4和IPv6双栈

**功能状态**：`Kubernetes v1.16 [alpha]`

IPv4/IPv6双栈技术可以给[Pod](../业务组件/泡德（Pod）/Pod.md)和[Service](Service.md)分配IPv4和IPv6地址。

如果你为集群开启了双栈网络，集群就可以同时分配IPv4和IPv6地址了。

## 支持的特性

集群开启IPv4/IPv6双栈可以提供以下特性：

- 双栈Pod网络（每个Pod得到一个IPv4和IPv6地址）
- 为Service开启IPv4和IPv6（每个Service同时只能使用一个协议族）
- Pod的集群出口路由（比如互联网）可以同时基于IPv4和IPv6接口进行

## 前提条件

k8s集群要想使用IPv4/IPv6双栈：

- k8s版本大于等于1.16
- 底层要支持双栈（云服务商或其他什么玩意，必须为k8s节点提供可路由的IPv4/IPv6网络接口）
- 支持双栈的网络插件（比如Kubenet或Calico）

## 开启IPv4/IPv6双栈

要想开启双栈，必须要为相关组件打开`IPv6DualStack`[特性门](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)，设置相关网络参数：

- kube-apiserver：
  - `--feature-gates="IPv6DualStack=true"`
- kube-controller-manager：
  - `--feature-gates="IPv6DualStack=true"`
  - `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
  - `--service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>`
  - `--node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6`，IPv4默认是/24，IPv6默认是/64
- kubelet：
  - `--feature-gates="IPv6DualStack=true"`
- kube-proxy：
  - `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
  - `--feature-gates="IPv6DualStack=true"`

>**注意**：
>IPv4 CIDR示例：`10.244.0.0/16`（你也可以自己定）
>
>IPv6 CIDR示例：`fdXY:IJKL:MNOP:15::/64`（这里只是给出格式，并不是有效的地址，见[RFC 4193](https://tools.ietf.org/html/rfc4193)）

## Service

开启双栈后，你就可以创建带有IPv4或IPv6地址的[Service](Service.md)了。通过`.spec.ipFamily`字段为Service的cluster IP来选择协议族。只能在新建Service的时候才能设置这个字段。`.spec.ipFamily`字段是可选的，只有你计划要开启IPv4和IPv6的[Service](Service.md)和[Ingress](Ingress.md)的时候才应该去设置它。对于[出口流量](#出口流量)则不是必须要设置这个属性。

>**注意**：集群的默认地址族是根据kube-controller-manager中配置的第一个`--service-cluster-ip-range`来定的。

可以将`.spec.ipFamily`设置为：

- `IPv4`：api server会在`service-cluster-ip-range`中分配一个`ipv4`的IP
- `IPv6`：api server会在`service-cluster-ip-range`中分配一个`ipv6`的IP

下面的Service定义中不包含`ipFamily`属性。k8s会根据第一个`service-cluster-ip-range`来给Service分配一个IP（也叫做“集群IP（cluster IP）”）。

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

下面的Servcie定义中包含了`ipFamily`属性。k8s会根据配置的`service-cluster-ip-range`为Service分配一个IPv6地址（也叫做“集群IP”（cluster IP））。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ipFamily: IPv6
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

作为对比，下面的Service定义会根据`service-cluster-ip-range`来分配一个IPv4的地址（也叫做“集群IP”（cluster IP））。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ipFamily: IPv4
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

### LoadBalancer类型

如果云服务商支持IPv6，开启了外部负载均衡器，除了要把`ipFamily`设置为`IPv6`，还要把`type`设置为`LoadBalancer`。

## 出口流量

可以使用可路由的公网或非公网的IPv6地址，底层的[CNI](../计算，存储与网络拓展/网络插件.md#cni)程序可以实现这种传输。如果你的Pod用了一个非公网可路由的IPv6地址，然后想访问集群外部（比如互联网），那你必须对出口流量和返回内容做IP伪装。[ip-masq-agent](https://github.com/kubernetes-sigs/ip-masq-agent)可以支持双栈，所以在双栈的集群中你可以使用ip-masq-agent来进行IP伪装。

## 已知问题

- Kubenet强制IPv4和IPv6的IP位置报告（--cluster-cidr）。才疏学浅，不知道这是啥意思。

## 截下来……

- [验证IPv4/IPv6双栈网络](https://kubernetes.io/docs/tasks/network/validate-dual-stack/)