# Ingress Controller

为了让Ingress能够正常的工作，集群中必须要有Ingress Controller。

其他类型的控制器都是作为`kube-controller-manager`的一部分来运行的，而Ingress Controller并不是随着集群自动启动的。通过本文的学习，选择最适合你的Ingress Controller实现。

k8s本身目前负责维护[GCE](https://github.com/kubernetes/ingress-gce/blob/master/README.md)和[Nginx](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)的Controller。

## 其他的控制器

- [AKS Application Gateway Ingress Controller](https://github.com/Azure/application-gateway-kubernetes-ingress)，通过[Azure Application Gateway](https://docs.microsoft.com/zh-cn/azure/application-gateway/overview)来启用[AKS集群](https://docs.microsoft.com/zh-cn/azure/aks/kubernetes-walkthrough-portal)中的Ingress。
- [Ambassador](https://www.getambassador.io/) API网关，这是[Datawire](https://www.getambassador.io/?utm_source=https://www.datawire.io/)提供的基于[Envoy](https://www.envoyproxy.io/)的Ingress Controller，包含了[社区](https://www.getambassador.io/docs/latest/)和[商业版](https://www.getambassador.io/pro/)的支持。
- [AppsCode Inc.](https://appscode.com/)提供了基于最常用的[HAProxy](https://www.haproxy.org/)的Ingress Controller [Voyager](https://voyagermesh.com/)。
- [AWS ALB Ingress Controller](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)，开启在[AWS Application LoadBalancer](https://aws.amazon.com/cn/elasticloadbalancing/)下的Ingress。
- [Contour](https://projectcontour.io/)，这是VMware提供的基于[Envoy](https://www.envoyproxy.io/)的Ingress Controller。
- Citrix为它的硬件（MPX）产品、虚拟化（VPX）产品、[云端](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/deployment)产品、基于[裸金属](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/deployment/baremetal)的[免费容器化（CPX）ADC](https://www.citrix.com/products/citrix-adc/cpx-express.html)产品，都提供了一个[Ingress Controller](https://github.com/citrix/citrix-k8s-ingress-controller)。
- F5 Networks提供了对[F5 BIG-IP 容器Ingress Service](https://clouddocs.f5.com/containers/latest/userguide/kubernetes/)的[支持和维护](https://support.f5.com/csp/article/K86859508)。
- [Gloo](https://docs.solo.io/gloo/latest/)是一个开源的Ingress Controller，同样基于[Envoy](https://www.envoyproxy.io/)，提供API网关的能力，在[solo.io](https://www.solo.io/)上提供企业版的支持。
- [HAProxy Ingress](https://haproxy-ingress.github.io/)，这是个可定制程度非常高的Ingress Controller，由社区驱动，基于HAProxy。
- [HAProxy Ingress Controller For Kubernetes](https://github.com/haproxytech/kubernetes-ingress)，这是由[HAProxy Technologies](https://www.haproxy.com/)支持和维护的。见[官方文档](https://www.haproxy.com/documentation/hapee/1-9r1/installation/kubernetes-ingress-controller/)。
- [Control Ingress Traffic](https://istio.io/latest/docs/tasks/traffic-management/ingress/)，基于[Istio](https://istio.io/)的Ingress Controller。
- [Kong Ingress Controller for Kubernetes](https://github.com/Kong/kubernetes-ingress-controller)，由[Kong](https://konghq.com/)提供[社区](https://discuss.konghq.com/c/kubernetes/19)或[商业版](https://konghq.com/products/kong-enterprise/)的支持和维护。
- [NGINX Ingress Controller for Kubernetes](https://www.nginx.com/products/nginx/kubernetes-ingress-controller)，由[NGINX, Inc](https://www.nginx.com/)提供支持和维护。
- [Skipper](https://opensource.zalando.com/skipper/kubernetes/ingress-controller/)，一套HTTP路由和反向代理的组合，包含了k8s中的Ingress支持，这是一套程序库，用于让你自己构建自定义的代理机制。
- [Traefik](https://github.com/containous/traefik)，功能完备的Ingress Controller（包含[Let's Encrypt](https://letsencrypt.org/)、秘钥、http2、websocket），还有来自[Containous](https://containo.us/services)的商业版支持。

## 使用多个Ingress Controller

一个集群里头可以部署[任意多个Ingress Cotnroller](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/multiple-ingress.md#multiple-ingress-controllers)。当你创建一个Ingress的时候，应该给它们打上合适的[`ingress.class`](https://github.com/kubernetes/ingress-gce/blob/master/docs/faq/README.md#how-do-i-run-multiple-ingress-controllers-in-the-same-cluster)注解，表明应该用哪个Ingress Controller。

如果不定义它的类，那云服务商应该会提供一个默认的Ingress Controller。

理想情况下所有的Ingress Controller都可以满足这里的定义，但它们彼此的具体使用又稍有不同。

>**注意**：选择一个Ingress Controller之前一定要好好看一下它的文档。

## 下一步……

- 学习[Ingress](Ingress.md).
- [在Minikube上面用NGINX Controller创建Ingress](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)。