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
- [服务发布（ServiceType）](#服务发布（ServiceType）)
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

### 不带选择器的Service

## 多端口

## 自定义IP

## 服务发现

## Headless Service

## 服务发布ServiceType

## 缺陷

## 虚拟IP的实现

## API对象

## 支持的协议

## 下一步……