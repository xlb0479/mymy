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

### 创建DaemonSet