# DaemonSet

*DaemonSet*可以确保所有（或部分）节点都会运行Pod的一个副本。节点加入集群的时候Pod就会加到节点上。节点从集群删除时Pod也随即被回收。删除DaemonSet时会清理由它创建的Pod。

典型案例如下：

- 在每个节点上运行集群存储的守护进程，比如`glusterd`、`ceph`。
- 为每个节点设置日志收集程序，比如`fluentd`、`filebeat`。
- 给每个节点加监控程序，比如[Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、[Flowmill](https://github.com/Flowmill/flowmill-k8s/)、[Sysdig Agent](https://docs.sysdig.com/?lang=en)、`collectd`、[Dynatrace OneAgent](https://www.dynatrace.com/technologies/kubernetes-monitoring/)、[AppDynamics Agent](https://docs.appdynamics.com/display/CLOUD/Container+Visibility+with+Kubernetes)、[Datadog agent](https://docs.datadoghq.com/agent/kubernetes/?tab=helm)、[New Relic agent](https://docs.newrelic.com/docs/integrations/kubernetes-integration/installation/kubernetes-installation-configuration)、Ganglia的`gmond`、[Instana Agent](https://www.instana.com/supported-integrations/kubernetes-monitoring/)以及[Elastic Metricbeat](https://www.elastic.co/guide/en/beats/metricbeat/current/running-on-kubernetes.html)。

再比如简单一些的用法，一个DaemonSet，覆盖所有集群，用于各种守护进程。又比如复杂一点的用法，为一种守护进程建多个DaemonSet，每个DaemonSet有不同的选项、对不同硬件有不同的内存和CPU请求。

- [写一个DaemonSet](#写一个DaemonSet)
- [如何调度](#如何调度)
- [与守护进程Pod通信](#与守护进程Pod通信)
- [更新DaemonSet](#更新DaemonSet)
- [替代方案](#替代方案)

## 写一个DaemonSet

### 创建一个DaemonSet

可以使用YAML文件来定义DaemonSet。比如下面这个`daemonset.yaml`，定义了一个运行fluentd-elasticsearch镜像的DaesmonSet。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

使用YAML文件创建DaemonSet：

```shell script
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

### 必填字段

跟其他的Kubernetes配置一样，DaemonSet需要`apiVersion`、`kind`、和`metadata`字段。关于大部分配置的用法，可以看[应用部署]()、[配置容器]()以及[使用kubectl管理对象](../../概要/Kubernetes对象/k8s对象管理.md)。

DaemonSet对象的名字必须是有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

DaemonSet必须要有[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)属性。

### Pod模板

`.spec.template`是`.spec`的必填项之一。

`.spec.template`代表一个[Pod模板](../泡德（Pod）/概要.md#Pod模板)。格式和[Pod](../泡德（Pod）/Pod.md)一样，只不过它嵌在别的属性里了，并且没有`apiVersion`和`kind`。

除了Pod基本的必填项以外，DaemonSet中的Pod模板必须要定义好合适的标签（见[Pod选择器](#Pod选择器)）。

DaemonSet中的Pod模板必须将[`RestartPolicy`](../泡德（Pod）/Pod生命周期.md#重启策略)设置为`Always`，或者不填，默认就是`Always`。

### Pod选择器

`.spec.selector`字段是一个Pod选择器。它的作用跟[Job](Job.md)里的`.spec.selector`一样。

从1.8版本开始，Pod选择器必须跟`.spec.template`中的标签匹配。如果不填的话不会给你加任何默认值。默认选择器跟`kubectl apply`命令无法兼容。并且在DaemonSet创建之后它的`.spec.selector`是不能改的。修改选择器可能导致一些Pod变成无人认领的孤儿，让用户感到非常的费解，困惑，以至于拿头撞墙。

`.spec.selector`包含两个字段：

- `matchLabels`-作用同[ReplicationController](ReplicationController.md)中的`.spec.selector`。
- `matchExpression`-通过各种key、value列表、操作符创建更复杂的选择器。

两种用法一起用的时候，结果取交集。

如果定义了`.spec.selector`，则它必须跟`.spec.template.metadata.labels`匹配。否则API报错。

同样，不要创建可能被这些选择器匹配到的普通的Pod，不要直接创建，也不要用其他的DaemonSet创建，也不要用其他的组件比如ReplicaSet创建。否则DaemonSet[控制器](../../集群架构/控制器.md)会认为这是它自己建的。k8s不会阻止你这么干。如果非要这么干，我能想到的场景就是为了测试某个节点的什么功能，仅仅是为了测试。

### 只在个别节点上运行Pod

如果你定义了`.spec.template.spec.nodeSelector`，DaemonSet控制器只会在跟[节点选择器]()匹配的节点上创建Pod。同样，如果你定义了`.spec.template.spec.affinity`，DaemonSet控制器只会在跟[节点亲和性]()匹配的节点上创建Pod。如果这两个字段都没有，DaemonSet控制器会在所有节点上创建Pod。

## 如何调度

### 由默认调度器调度

**功能状态**：`Kubernetes v1.18 [stable]`

DaemonSet可以确保所有适当的节点都会运行一个Pod副本。一般来说，Pod运行在哪个节点上是归k8s调度器管的。但是DaemonSet的Pod是由DaemonSet控制器来创建和调度的。这就导致了如下问题：

- 不一致的Pod行为：普通Pod创建完等待调度的状态是`Pending`，但DaemonSet的Pod建完之后可不是`Pending`状态。这让用户感到困扰，焦虑。
- [Pod抢占机制]()是由默认调度器处理的。抢占机制开启的情况下，DaemonSet控制器在调度的过程中并不会考虑到Pod的优先级和抢占问题。

开启`ScheduleDaemonSetPods`[特性门]()可以让你用默认调度器来调度DaemonSet，而不是DaemonSet控制器，此时可以为DaemonSet的Pod设置`NodeAffinity`，而不用`.spec.nodeName`。然后默认调度器就会将Pod绑定到目标主机上。如果DaemonSet的Pod已经设置了节点亲和性则会被替换掉。DaemonSet只有在创建或修改Pod的时候才会这么做，并且不会对`.spec.template`产生任何更改。

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - target-host-name
```

此外，会自动为DaemonSet的Pod添加`node.kubernetes.io/unschedulable:NoSchedule`这个冷屁股。默认调度器在调度的时候会忽略那些`unschedulable`的节点。

### 冷屁股（Taint）和热脸（Toleration）

尽管DaemonSet的Pod遵守[热脸和冷屁股]()规则，基于相关特性考虑，以下热脸会自动加到DaemonSet的Pod上。

Key|效果|版本|描述
-|-|-|-
`node.kubernetes.io/not-ready`|NoExecute|1.13+|当节点出现问题，比如网络异常，DaemonSet的Pod不会被踢掉。
`node.kubernetes.io/unreachable`|NoExecute|1.13+|当节点出现问题，比如网络异常，DaemonSet的Pod不会被踢掉。
`node.kubernetes.io/disk-pressure`|NoSchedule|1.8+|
`node.kubernetes.io/memory-pressure`|NoSchedule|1.8+|
`node.kubernetes.io/unschedulable`|NoSchedule|1.12+|默认调度器会容忍不可调度属性。
`node.kubernetes.io/network-unavailable`|NoSchedule|1.12+|DaemonSet的Pod使用主机网络，默认调度器会容忍网络不可用的属性。