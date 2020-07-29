# 通过Service来连接应用

## k8s中的容器连接模式

此时你应该已经有一个跑起来的分布式应用了，你可以把它暴露到一个网络中。在我们讨论k8s的网络思想之前，有必要先看看如何用“普通的”方式基于Docker进行网络连接。

默认情况下Docker会使用一个主机私有的网络，所以容器只能跟同一个主机的容器进行通信。Docker容器要想跨节点通信，那就必须再主机自己的IP地址上进行端口分配，然后再转发或代理到容器上。这就意味着，容器调度的时候还要规划端口的调度，或者是进行动态端口分配。

如果要在多个开发者或团队中进行端口的分配规划，这样做的可扩展性是很差的，而且让整件事儿不那么透明，用户要知道这些集群相关的事儿。k8s则认为Pod是可以跟其他Pod通信的，不管它们在哪个主机上。k8s给每个Pod分配它在集群内部的私有IP，因此你不用明确地去创建Pod之间的连接或者是容器端口和主机端口的映射。这就意味着，Pod内部的所有容器可以通过localhost访问对方的端口，集群中的所有Pod在没有NAT的情况下就能看到彼此。本文剩余的部分会详细介绍如果在这种网络模型中运行可靠的服务。

为了验证方便，这里要使用一个Nginx服务器进行演示。

## 将Pod暴露到集群中

这个栗子我们前面见过，我们再拿到这里，这次我们重点关注网络部分。创建一个nginx的Pod，注意它有一个容器端口的定义：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

这样，在集群中任何节点都可以访问到了。查看Pod所在的节点：

```shell script
kubectl apply -f ./run-my-nginx.yaml
kubectl get pods -l run=my-nginx -o wide
```

```text
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
my-nginx-3800858182-jr4a2   1/1       Running   0          13s       10.244.3.4    kubernetes-minion-905m
my-nginx-3800858182-kna2y   1/1       Running   0          13s       10.244.2.5    kubernetes-minion-ljyd
```

查看Pod的IP：

```shell script
kubectl get pods -l run=my-nginx -o yaml | grep podIP
    podIP: 10.244.3.4
    podIP: 10.244.2.5
```

你可以ssh到任意一个节点上curl它们的IP。注意容器*并没有*使用节点的80端口，而且也没有任何特殊的NAT规则把流量路由到Pod。这就意味着你可以在同一个节点上运行多个相同containerPort的Nginx的Pod，而且还可以用IP地址从任意的Pod或节点上访问到它。和Docker类似，端口依旧是支持映射到节点的网络接口上的，但是由于这种网络模型的出现，这种需求基本上没有了。

如果你特好奇，跟个猫似的，你可以去看看[这到底是怎么实现的](../集群管理/集群网络.md)。

## 创建一个Service

此时，我们有了两个Nginx运行在扁平的、集群范围的地址空间中。理论上来说，你可以直接跟Pod进行通信，但如果节点挂了怎么办？Pod跟它同归于尽了，Deployment会创建新的Pod，但是IP不同了。这就是Service要解决的问题。

一个k8s的Service就是一个对一组运行在集群中的Pod的逻辑抽象，这些Pod提供完全相同的功能。创建之后，每个Service会得到一个唯一的IP地址（也叫clusterIP）。这个IP地址是跟这个Service的一生绑定在一起的，只要Service在，它就不会变。Pod可以跟Service进行通信，流量会自动被Service在它的Pod之间进行负载均衡。

你可以用`kubectl expose`给这两个Nginx创建一个Service：

```shell script
kubectl expose deployment/my-nginx
```

```text
service/my-nginx exposed
```

效果跟用`kubectl apply -f`执行下面的yaml文件是一样的：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

这个定义会创建一个Service，指向所有带`run: my-nginx`标签的Pod的80端口，暴露到一个抽象的Service端口上（`targetPort`：容器接收流量的端口，`port`：抽象Service端口，可以是任意端口，就是为了让其他Pod来访问这个Service）。关于Service支持的属性可以去看[Service](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#service-v1-core) API对象。检查一下我们创建的Service：

```shell script
kubectl get svc my-nginx
```

```text
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.0.162.149   <none>        80/TCP    21s
```

前文书我们讲过，一个Service后面是一组Pod。这些Pod是通过`endpoint`暴露的。Service的选择器会持续进行计算，将结果POST到同样名为`my-nginx`的Endpoint对象上。当一个Pod挂了，会自动从endpoint中移除。来看一下endpoint，注意它们的IP跟第一步创建的Pod的IP是一样的：

```shell script
kubectl describe svc my-nginx
```

```text
Name:                my-nginx
Namespace:           default
Labels:              run=my-nginx
Annotations:         <none>
Selector:            run=my-nginx
Type:                ClusterIP
IP:                  10.0.162.149
Port:                <unset> 80/TCP
Endpoints:           10.244.2.5:80,10.244.3.4:80
Session Affinity:    None
Events:              <none>
```

```shell script
kubectl get ep my-nginx
```

```text
NAME       ENDPOINTS                     AGE
my-nginx   10.244.2.5:80,10.244.3.4:80   1m
```

此时，你就可以用curl通过`<CLUSTER-IP>:<PORT>`看一下这个Nginx的Service了，集群中的任意一个节点都可以。注意这个Service的IP完全是虚拟的，绝对不会接触底层物理网络。如果你特好奇，那你可以去看一下[服务代理](Service.md)。

## 访问Service

k8s支持两种主要的模式来找到一个Service——环境变量和DNS。前者开箱即用，后者需要[CoreDNS插件](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/coredns)。

>**注意**：如果环境变量不足以满足需求（变量冲突、变量过多、只用DNS等）你可以在[Pod定义](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#pod-v1-core)时将`enableServiceLinks`设置为`false`来关闭这种模式。

### 环境变量

当一个Pod运行在节点上，kubelet会将每个活动的Service相关的环境变量注入到Pod中。这就导致了一个顺序问题。想知道为什么？检查一下你的Nginx的Pod的环境变量（你的实际Pod名字可能会不一样）：

```shell script
kubectl exec my-nginx-3800858182-jr4a2 -- printenv | grep SERVICE
```

```text
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```

注意，这里并没有你刚才创建的Service。这是因为Pod是在Service之前创建的。另一个问题是，调度器可能会把两个Pod调度到同一台机器上，如果这个机器挂了，那你的Service整个就完犊子了。我们可以把这两个Pod都杀掉，等Deployment进行重建。这次，Service就在Pod*之前*了。这就可以为Service提供在调度器层面的Pod分布控制（每个节点的容量都保持一致），也就会有正确的环境变量了：

```shell script
kubectl scale deployment my-nginx --replicas=0; kubectl scale deployment my-nginx --replicas=2;

kubectl get pods -l run=my-nginx -o wide
```

```text
NAME                        READY     STATUS    RESTARTS   AGE     IP            NODE
my-nginx-3800858182-e9ihh   1/1       Running   0          5s      10.244.2.7    kubernetes-minion-ljyd
my-nginx-3800858182-j4rm4   1/1       Running   0          5s      10.244.3.8    kubernetes-minion-905m
```

你会发现Pod的名字变了，因为它们被杀掉重建了。

```shell script
kubectl exec my-nginx-3800858182-e9ihh -- printenv | grep SERVICE
```

```text
KUBERNETES_SERVICE_PORT=443
MY_NGINX_SERVICE_HOST=10.0.162.149
KUBERNETES_SERVICE_HOST=10.0.0.1
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_PORT_HTTPS=443
```

### DNS

k8s提供了一个集群DNS插件Service，自动为其他Service生成DNS记录。我们来检查一下当前的状况：

```shell script
kubectl get services kube-dns --namespace=kube-system
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   8m
```

下面的内容假设你已经有一个稳定的Service了（my-nginx），并且DNS服务已经给它分配了一个记录。我们这里用的是CoreDNS集群插件（应用名为`kube-dns`），因此，你可以在集群中的任意Pod中使用标准方法（比如`gethostbyname()`）来访问到这个Service。如果CoreDNS没有运行，可以参考[CoreDNS README](https://github.com/coredns/deployment/tree/master/kubernetes)或[安装CoreDNS](https://kubernetes.io/docs/tasks/administer-cluster/coredns/#installing-coredns)来开启它。我们来运行一个curl应用进行测试：

```shell script
kubectl run curl --image=radial/busyboxplus:curl -i --tty
```

```text
Waiting for pod default/curl-131556218-9fnch to be running, status is Pending, pod ready: false
Hit enter for command prompt
```

好了，我们执行一下`nslookup my-nginx`：

```text
[ root@curl-131556218-9fnch:/ ]$ nslookup my-nginx
Server:    10.0.0.10
Address 1: 10.0.0.10

Name:      my-nginx
Address 1: 10.0.162.149
```

## Service安全

目前我们都是在集群内访问Service。将Service暴露到互联网之前，你需要确认通信的安全。你需要：

- https的自签名证书（除非你已经有证书了）
- 配置Nginx使用这个证书
- 创建[Secret](../配置/Secret.md)让Pod可以获取到这个证书。

你可以参考[Nginx Https用例](https://github.com/kubernetes/examples/tree/master/staging/https-nginx/)来实现这些。这里面需要安装Go和Make工具。如果你不想装，可以参考后面的手动操作。简单来说就是：

```shell script
make keys KEY=/tmp/nginx.key CERT=/tmp/nginx.crt
kubectl create secret tls nginxsecret --key /tmp/nginx.key --cert /tmp/nginx.crt
```

```text
secret/nginxsecret created
```

```shell script
kubectl get secrets
```

```text
NAME                  TYPE                                  DATA      AGE
default-token-il9rc   kubernetes.io/service-account-token   1         1d
nginxsecret           kubernetes.io/tls                     2         1m
```

还有ConfigMap：

```shell script
kubectl create configmap nginxconfigmap --from-file=default.conf
```

```text
configmap/nginxconfigmap created
```

```shell script
kubectl get configmaps
```

```text
NAME             DATA   AGE
nginxconfigmap   1      114s
```

如果你运行Make有问题（比如在Windows上），下面是手动操作流程：

```shell script
# Create a public private key pair
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /d/tmp/nginx.key -out /d/tmp/nginx.crt -subj "/CN=my-nginx/O=my-nginx"
# Convert the keys to base64 encoding
cat /d/tmp/nginx.crt | base64
cat /d/tmp/nginx.key | base64
```

用上一步的输出内容来创建一个yaml文件。Base64产生的编码值应该都放在一行里。

```yaml
apiVersion: "v1"
kind: "Secret"
metadata:
  name: "nginxsecret"
  namespace: "default"
type: kubernetes.io/tls
data:
  tls.crt: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURIekNDQWdlZ0F3SUJBZ0lKQUp5M3lQK0pzMlpJTUEwR0NTcUdTSWIzRFFFQkJRVUFNQ1l4RVRBUEJnTlYKQkFNVENHNW5hVzU0YzNaak1SRXdEd1lEVlFRS0V3aHVaMmx1ZUhOMll6QWVGdzB4TnpFd01qWXdOekEzTVRKYQpGdzB4T0RFd01qWXdOekEzTVRKYU1DWXhFVEFQQmdOVkJBTVRDRzVuYVc1NGMzWmpNUkV3RHdZRFZRUUtFd2h1CloybHVlSE4yWXpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSjFxSU1SOVdWM0IKMlZIQlRMRmtobDRONXljMEJxYUhIQktMSnJMcy8vdzZhU3hRS29GbHlJSU94NGUrMlN5ajBFcndCLzlYTnBwbQppeW1CL3JkRldkOXg5UWhBQUxCZkVaTmNiV3NsTVFVcnhBZW50VWt1dk1vLzgvMHRpbGhjc3paenJEYVJ4NEo5Ci82UVRtVVI3a0ZTWUpOWTVQZkR3cGc3dlVvaDZmZ1Voam92VG42eHNVR0M2QURVODBpNXFlZWhNeVI1N2lmU2YKNHZpaXdIY3hnL3lZR1JBRS9mRTRqakxCdmdONjc2SU90S01rZXV3R0ljNDFhd05tNnNTSzRqYUNGeGpYSnZaZQp2by9kTlEybHhHWCtKT2l3SEhXbXNhdGp4WTRaNVk3R1ZoK0QrWnYvcW1mMFgvbVY0Rmo1NzV3ajFMWVBocWtsCmdhSXZYRyt4U1FVQ0F3RUFBYU5RTUU0d0hRWURWUjBPQkJZRUZPNG9OWkI3YXc1OUlsYkROMzhIYkduYnhFVjcKTUI4R0ExVWRJd1FZTUJhQUZPNG9OWkI3YXc1OUlsYkROMzhIYkduYnhFVjdNQXdHQTFVZEV3UUZNQU1CQWY4dwpEUVlKS29aSWh2Y05BUUVGQlFBRGdnRUJBRVhTMW9FU0lFaXdyMDhWcVA0K2NwTHI3TW5FMTducDBvMm14alFvCjRGb0RvRjdRZnZqeE04Tzd2TjB0clcxb2pGSW0vWDE4ZnZaL3k4ZzVaWG40Vm8zc3hKVmRBcStNZC9jTStzUGEKNmJjTkNUekZqeFpUV0UrKzE5NS9zb2dmOUZ3VDVDK3U2Q3B5N0M3MTZvUXRUakViV05VdEt4cXI0Nk1OZWNCMApwRFhWZmdWQTRadkR4NFo3S2RiZDY5eXM3OVFHYmg5ZW1PZ05NZFlsSUswSGt0ejF5WU4vbVpmK3FqTkJqbWZjCkNnMnlwbGQ0Wi8rUUNQZjl3SkoybFIrY2FnT0R4elBWcGxNSEcybzgvTHFDdnh6elZPUDUxeXdLZEtxaUMwSVEKQ0I5T2wwWW5scE9UNEh1b2hSUzBPOStlMm9KdFZsNUIyczRpbDlhZ3RTVXFxUlU9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  tls.key: "LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRQ2RhaURFZlZsZHdkbFIKd1V5eFpJWmVEZWNuTkFhbWh4d1NpeWF5N1AvOE9ta3NVQ3FCWmNpQ0RzZUh2dGtzbzlCSzhBZi9WemFhWm9zcApnZjYzUlZuZmNmVUlRQUN3WHhHVFhHMXJKVEVGSzhRSHA3VkpMcnpLUC9QOUxZcFlYTE0yYzZ3MmtjZUNmZitrCkU1bEVlNUJVbUNUV09UM3c4S1lPNzFLSWVuNEZJWTZMMDUrc2JGQmd1Z0ExUE5JdWFubm9UTWtlZTRuMG4rTDQKb3NCM01ZUDhtQmtRQlAzeE9JNHl3YjREZXUraURyU2pKSHJzQmlIT05Xc0RadXJFaXVJMmdoY1kxeWIyWHI2UAozVFVOcGNSbC9pVG9zQngxcHJHclk4V09HZVdPeGxZZmcvbWIvNnBuOUYvNWxlQlkrZStjSTlTMkQ0YXBKWUdpCkwxeHZzVWtGQWdNQkFBRUNnZ0VBZFhCK0xkbk8ySElOTGo5bWRsb25IUGlHWWVzZ294RGQwci9hQ1Zkank4dlEKTjIwL3FQWkUxek1yall6Ry9kVGhTMmMwc0QxaTBXSjdwR1lGb0xtdXlWTjltY0FXUTM5SjM0VHZaU2FFSWZWNgo5TE1jUHhNTmFsNjRLMFRVbUFQZytGam9QSFlhUUxLOERLOUtnNXNrSE5pOWNzMlY5ckd6VWlVZWtBL0RBUlBTClI3L2ZjUFBacDRuRWVBZmI3WTk1R1llb1p5V21SU3VKdlNyblBESGtUdW1vVlVWdkxMRHRzaG9reUxiTWVtN3oKMmJzVmpwSW1GTHJqbGtmQXlpNHg0WjJrV3YyMFRrdWtsZU1jaVlMbjk4QWxiRi9DSmRLM3QraTRoMTVlR2ZQegpoTnh3bk9QdlVTaDR2Q0o3c2Q5TmtEUGJvS2JneVVHOXBYamZhRGR2UVFLQmdRRFFLM01nUkhkQ1pKNVFqZWFKClFGdXF4cHdnNzhZTjQyL1NwenlUYmtGcVFoQWtyczJxWGx1MDZBRzhrZzIzQkswaHkzaE9zSGgxcXRVK3NHZVAKOWRERHBsUWV0ODZsY2FlR3hoc0V0L1R6cEdtNGFKSm5oNzVVaTVGZk9QTDhPTm1FZ3MxMVRhUldhNzZxelRyMgphRlpjQ2pWV1g0YnRSTHVwSkgrMjZnY0FhUUtCZ1FEQmxVSUUzTnNVOFBBZEYvL25sQVB5VWs1T3lDdWc3dmVyClUycXlrdXFzYnBkSi9hODViT1JhM05IVmpVM25uRGpHVHBWaE9JeXg5TEFrc2RwZEFjVmxvcG9HODhXYk9lMTAKMUdqbnkySmdDK3JVWUZiRGtpUGx1K09IYnRnOXFYcGJMSHBzUVpsMGhucDBYSFNYVm9CMUliQndnMGEyOFVadApCbFBtWmc2d1BRS0JnRHVIUVV2SDZHYTNDVUsxNFdmOFhIcFFnMU16M2VvWTBPQm5iSDRvZUZKZmcraEppSXlnCm9RN3hqWldVR3BIc3AyblRtcHErQWlSNzdyRVhsdlhtOElVU2FsbkNiRGlKY01Pc29RdFBZNS9NczJMRm5LQTQKaENmL0pWb2FtZm1nZEN0ZGtFMXNINE9MR2lJVHdEbTRpb0dWZGIwMllnbzFyb2htNUpLMUI3MkpBb0dBUW01UQpHNDhXOTVhL0w1eSt5dCsyZ3YvUHM2VnBvMjZlTzRNQ3lJazJVem9ZWE9IYnNkODJkaC8xT2sybGdHZlI2K3VuCnc1YytZUXRSTHlhQmd3MUtpbGhFZDBKTWU3cGpUSVpnQWJ0LzVPbnlDak9OVXN2aDJjS2lrQ1Z2dTZsZlBjNkQKckliT2ZIaHhxV0RZK2Q1TGN1YSt2NzJ0RkxhenJsSlBsRzlOZHhrQ2dZRUF5elIzT3UyMDNRVVV6bUlCRkwzZAp4Wm5XZ0JLSEo3TnNxcGFWb2RjL0d5aGVycjFDZzE2MmJaSjJDV2RsZkI0VEdtUjZZdmxTZEFOOFRwUWhFbUtKCnFBLzVzdHdxNWd0WGVLOVJmMWxXK29xNThRNTBxMmk1NVdUTThoSDZhTjlaMTltZ0FGdE5VdGNqQUx2dFYxdEYKWSs4WFJkSHJaRnBIWll2NWkwVW1VbGc9Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K"
```

然后用这个文件来创建Secret：

```shell script
kubectl apply -f nginxsecrets.yaml
kubectl get secrets
```

```text
NAME                  TYPE                                  DATA      AGE
default-token-il9rc   kubernetes.io/service-account-token   1         1d
nginxsecret           kubernetes.io/tls                     2         1m
```

现在修改Nginx应用，使用保存在Secret中的证书来启动https服务，还有Service，暴露两个端口（80和443）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: nginxsecret
      - name: configmap-volume
        configMap:
          name: nginxconfigmap
      containers:
      - name: nginxhttps
        image: bprashanth/nginxhttps:1.0
        ports:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume
```

这一套煎饼中需要注意的地方：

- 包含了Deployment和Service。
- [Nginx服务](https://github.com/kubernetes/examples/blob/master/staging/https-nginx/default.conf)用80端口处理HTTP，用443端口处理HTTPS，对应的Service暴露了这两个端口。
- 每个容器通过挂载到`/etc/nginx/ssl`的数据卷来访问秘钥。这个需要在Nginx服务启动*之前*完成。

```shell script
kubectl delete deployments,svc my-nginx; kubectl create -f ./nginx-secure-app.yaml
```

此时你可以从任意节点来访问Nginx服务。

```shell script
kubectl get pods -o yaml | grep -i podip
    podIP: 10.244.3.5
node $ curl -k https://10.244.3.5
...
<h1>Welcome to nginx!</h1>
```

注意奥，我们执行curl的时候加了`-k`参数，这是因为我们不知道Pod中的Nginx的证书情况，所以我们让curl忽略CName异常的情况。通过创建一个Service，我们把证书中使用的CName和真实的Pod的DNS记录关联起来，在进行Service查询的时候就能用上了。我们在另一个Pod中测一下（为了简单，重用了同样的Secret，Pod访问Service的时候只需要nginx.crt）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deployment
spec:
  selector:
    matchLabels:
      app: curlpod
  replicas: 1
  template:
    metadata:
      labels:
        app: curlpod
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: nginxsecret
      containers:
      - name: curlpod
        command:
        - sh
        - -c
        - while true; do sleep 1; done
        image: radial/busyboxplus:curl
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
```

```shell script
kubectl apply -f ./curlpod.yaml
kubectl get pods -l app=curlpod
```

```text
NAME                               READY     STATUS    RESTARTS   AGE
curl-deployment-1515033274-1410r   1/1       Running   0          1m
```

```shell script
kubectl exec curl-deployment-1515033274-1410r -- curl https://my-nginx --cacert /etc/nginx/ssl/tls.crt
...
<title>Welcome to nginx!</title>
...
```

## 暴露服务

你可能希望你的某些应用将Service暴露到外部IP上。k8s支持两种方式：NodePort和LoadBalancer。上一节的Service已经使用了`NodePort`，所以你的节点如果有公网IP，那你的Nginx已经可以处理来自互联网的HTTPS流量了。

```shell script
kubectl get svc my-nginx -o yaml | grep nodePort -C 5
  uid: 07191fb3-f61a-11e5-8ae5-42010af00002
spec:
  clusterIP: 10.0.162.149
  ports:
  - name: http
    nodePort: 31704
    port: 8080
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 32453
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    run: my-nginx
```

```shell script
kubectl get nodes -o yaml | grep ExternalIP -C 1
    - address: 104.197.41.11
      type: ExternalIP
    allocatable:
--
    - address: 23.251.152.56
      type: ExternalIP
    allocatable:
...

$ curl https://<EXTERNAL-IP>:<NODE-PORT> -k
...
<h1>Welcome to nginx!</h1>
```

现在我们用云上的负载均衡重建Service，只需要把`my-nginx`Service的`Type`从`NodePort`改成`LoadBalancer`：

```shell script
kubectl edit svc my-nginx
kubectl get svc my-nginx
```

```text
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP        PORT(S)               AGE
my-nginx   LoadBalancer   10.0.162.149     xx.xxx.xxx.xxx     8080:30163/TCP        21s
```

```shell script
curl https://<EXTERNAL-IP> -k
...
<title>Welcome to nginx!</title>
```

`EXTERNAL-IP`列的IP地址就是公网IP。`CLUSTER-IP`只能用在集群/内网环境中。

如果是在AWS上，`LoadBalancer`类型会创建一个ELB，使用的是一个（长长的）主机名而不是IP。因为太长了，用`kubectl get svc`输出的时候会出现格式问题，实际上你需要用`kubectl describe service my-nginx`来看到它。比如：

```shell script
kubectl describe service my-nginx
...
LoadBalancer Ingress:   a320587ffd19711e5a37606cf4a74574-1142138393.us-east-1.elb.amazonaws.com
...
```

## 下一步……

- [在集群中用Service来访问应用](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)
- [用Service连接前后端](https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/)
- [创建外部负载均衡](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)