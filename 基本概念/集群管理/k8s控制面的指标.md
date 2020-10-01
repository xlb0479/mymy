# k8s控制面的指标

系统组件的指标（metric）可以让你更好的观察到其内部正在发生的事情。而且指标对于构建dashboard和告警机制也非常有用。

k8s控制面（control plane）的指标以[Prometheus格式](https://prometheus.io/docs/instrumenting/exposition_formats/)暴露出来的，具备可读性。

## k8s指标

大部分情况下都可以通过HTTP服务的`/metrics`接口来获取指标。对于默认没有暴露接口的组件，可以通过`--bind-address`参数来开启。

相关组件的栗子：

- [kube-controller-manager](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
- [kube-proxy](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
- [kube-apiserver](../概要/Kubernetes组成.md#kube-apiserver)
- [kube-scheduler](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
- [kubelet](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)

在生产环境中，你可能会部署[Prometheus](https://prometheus.io/)或者其他的指标收集服务，周期性的采集指标并保存到某种时序数据库中。

注意[kubelet](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)还会在`/metrics/cadvisor`、`/metrics/resource`和`/metrics/probes`接口上暴露一些指标。这些指标的生命周期都不一样。

如果你的集群使用了[RBAC](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)，读取指标的时候需要通过用户、组或者ServiceAccount进行授权，必须要包含能够访问`/metrics`的ClusterRole。比如：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
```

## 指标生命周期

Alpha指标 → Stable指标 → 废弃指标 → 隐藏指标 → 删除

Alpha指标没有稳定性保障；随时可能被修改或者删除。

Stable指标可以保证不会发生变动；特别地，稳定性是指：

- 指标本身不会被删除（或重命名）
- 指标类型不会更改

废弃指标说明这个指标最终是要被删掉的；要查到相关版本，需要去看注解，其中会说明哪个k8s版本会将该指标废弃。

废弃前：

```text
# HELP some_counter this counts things
# TYPE some_counter counter
some_counter 0
```

废弃后：

```text
# HELP some_counter (Deprecated since 1.15.0) this counts things
# TYPE some_counter counter
some_counter 0
```

如果指标默认被隐藏，就说明该指标不需要被采集。要想使用隐藏指标，需要更改相关集群组件的配置。

一旦指标被删除，就不会再暴露出来了。改配置也无济于事。

## 显示隐藏指标

上面我们说了，管理员可以通过命令行参数来开启隐藏指标。这一招是给那些错过上一个版本废弃指标迁移的管理员们一个机会，给他们一个逃生门。

`show-hidden-metrics-for-version`选项需要指定一个版本号，声明你要开启哪一个版本中的废弃指标。版本号格式为x.y，x是大版本，y是小版本。补丁版本不需要，即便指标可以在补丁版本中被废弃，理由是指标废弃策略是基于小版本来实现的。

该选项只能设置上一个小版本作为它的参数。当管理员给`show-hidden-metrics-for-version`设置了上一个小版本，对应版本中所有的隐藏指标都会被暴露出来。不能用太老的版本，因为如果允许的话，指标还废弃个屁啊。

以指标`A`为例，假设`A`在1.n中被废弃了。根据指标废弃策略，我们可以得出以下结论：

- 在`1.n`中，该指标废弃，但是默认可以显示。
- 在`1.n+1`中，该指标默认隐藏，可以通过`show-hidden-metrics-for-version=1.n`开启。
- 在`1.n+2`中，该指标应该从代码库中删掉了。没有任何机会再去翻盘了。

如果你正在从`1.12`升级到`1.13`，但是仍旧依赖`1.12`中废弃的指标`A`，可以通过`--show-hidden-metrics=1.12`命令行选项来开启它，但是要记得在升级到`1.14`之前删除关于该指标的依赖。

## 组件指标

### kube-controller-manager指标

Controller manager的指标能够提供关于它自身的非常重要的性能和健康信息。其中包括常见的Go语言运行时指标，比如go_routine数量，以及控制器特定的指标比如etcd请求延迟或Cloudprovider（AWS、GCE、OpenStack）的API延迟，可以用来评估集群的健康状况。

从1.7版本开始，在GCE、AWS、Vsphere以及OpenStack上可以为存储操作提供详细的Cloudprovider指标。这些指标可以用来监控持久卷操作的健康状况。

比如在GCE上有这些指标：

```text
cloudprovider_gce_api_request_duration_seconds { request = "instance_list"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_insert"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_delete"}
cloudprovider_gce_api_request_duration_seconds { request = "attach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "detach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "list_disk"}
```

## 下一步……

- 了解[Prometheus的指标文本格式](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md#text-based-format)
- 查看[稳定的k8s指标](https://github.com/kubernetes/kubernetes/blob/master/test/instrumentation/testdata/stable-metrics-list.yaml)
- 学习[k8s废弃策略](https://v1-18.docs.kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-a-feature-or-behavior)