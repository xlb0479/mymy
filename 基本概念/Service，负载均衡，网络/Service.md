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

## Headless Service

## 服务发布（ServiceType）