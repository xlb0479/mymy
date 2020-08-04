# 通过HostAliases给Pod添加/etc/hosts记录

当DNS或者其他选项不适用的时候，可以给Pod的`/etc/hosts`添加记录，在Pod中对域名解析进行覆盖。

不建议使用非HostAliases的手段，因为这个文件是kubelet来管理的，有可能会在Pod创建或重启的时候被覆盖掉。

## 默认hosts文件

启动一个Nginx的Pod，得到一个IP地址：

```shell script
kubectl run nginx --image nginx
```

```text
pod/nginx created
```

查看Pod IP：

```shell script
kubectl get pods --output=wide
```

```text
NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
nginx    1/1       Running   0          13s    10.200.0.4   worker0
```

hosts文件的内容类似于这样：

```shell script
kubectl exec nginx -- cat /etc/hosts
```

```text
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.4	nginx
```

默认情况下`hosts`文件只会包含IPv4和IPv6的模板内容，比如`localhost`，以及它自己的主机名。

## 通过hostAliases添加额外记录

除了默认的模板，你可以在`hosts`文件中添加其他的记录。比如：想把`foo.local`、`bar.local`解析到`127.0.0.1`，把`foo.remote`、`bar.remote`解析到`10.1.2.3`，那你可以在Pod的`.spect.hostAliases`中配置主机别名：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox
    command:
    - cat
    args:
    - "/etc/hosts"
```

用这份配置启动一个Pod：

```shell script
kubectl apply -f https://k8s.io/examples/service/networking/hostaliases-pod.yaml
```

```text
pod/hostaliases-pod created
```

查看它的IPv4地址和状态：

```shell script
kubectl get pod --output=wide
```

```text
NAME                           READY     STATUS      RESTARTS   AGE       IP              NODE
hostaliases-pod                0/1       Completed   0          6s        10.200.0.5      worker0
```

`hosts`文件的内容类似这样：

```shell script
kubectl logs hostaliases-pod
```

```text
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.5	hostaliases-pod

# Entries added by HostAliases.
127.0.0.1	foo.local	bar.local
10.1.2.3	foo.remote	bar.remote
```

额外的记录加在最下面。

## 为什么kubelet要管理hosts文件？

让kubelet来[管理](https://github.com/kubernetes/kubernetes/issues/14633) Pod中每个容器的`hosts`文件，是为了阻止Docker在容器启动之后[修改](https://github.com/moby/moby/issues/17190)这个文件。

>**警告**：
>避免在容器中手动修改hosts文件。
>
>如果你手动改了，容器退出后改动内容就会消失。