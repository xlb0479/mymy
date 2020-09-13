# Pod安全策略

**功能状态**：`Kubernetes v1.18 [beta]`

Pod安全策略（Pod Security Policy）可以为Pod的创建和更新提供详细的授权。

## 什么是Pod安全策略？

*Pod安全策略*是一个集群层面的资源，它控制着Pod定义中对安全敏感的方面。[PodSecurityPolicy](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podsecuritypolicy-v1beta1-policy)对象定义了一组条件，Pod要想被系统接受，必须满足这些条件，以及其他相关的默认值。它允许管理员控制以下内容：

**控制面**|**字段名**
-|-
运行特权容器|[`privileged`](#privileged)
使用主机命名空间|[`hostPID`](#主机命名空间)、[`hostIPC`](#主机命名空间)
使用主机网络和端口|[`hostNetwork`](#主机命名空间)、[`hostPorts`](#主机命名空间)
使用数据卷类型|[`volumes`](#数据卷和文件系统)
使用主机文件系统|[`allowedHostPaths`](#数据卷和文件系统)
允许指定的FlexVolume驱动|[`allowedFlexVolumes`](#FlexVolume驱动)
为Pod的数据卷分配一个FSGroup|[`fsGroup`](#数据卷和文件系统)
需要使用只读的根文件系统|[`readOnlyRootFilesystem`](#数据卷和文件系统)
容器的用户和组ID|[`runAsUser`](#用户和组)、[`runAsGroup`](#用户和组)、[`supplementalGroups`](#用户和组)
限制提权到root|[`allowPrivilegeEscalation`](#提权)、[`defaultAllowPrivilegeEscalation`](#提权)
Linux Capabilities|[`defaultAddCapabilities`](#Capabilities)、[`requiredDropCapabilities`](#Capabilities)、[`allowedCapabilities`](#Capabilities)
容器的SELinux上下文|[`seLinux`](#SELinux)
允许的Proc挂载类型|[`allowedProcMountTypes`](#AllowedProcMountTypes)
容器的AppArmor配置|[注解](#AppArmor)
容器的seccomp配置|[注解](#Seccomp)
容器的sysctl配置|[`forbiddenSysctls`](#Sysctl)、[`allowedUnsafeSysctls`](#Sysctl)

## 开启Pod安全策略

Pod安全策略控制是作为一个可选的（但是建议的）[准入控制器](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecuritypolicy)实现的。通过[开启准入控制器](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-control-plug-in)来实现Pod安全策略，但如果仅仅是开启，没有授权任何策略，**是无法创建任何Pod的**。

由于Pod安全策略API（`policy/v1beta1/podsecuritypolicy`）是独立于准入控制器来开启的，所以对于现有集群，建议先添加策略并授权，然后再开启准入控制器。

## 策略授权

当创建了一个PodSecurityPolicy资源后，它什么也干不了。要想使用，用户或者目标Pod的[ServiceAccount](https://v1-18.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)比如通过授权才能使用这个策略，允许策略中的`use`动词。

大部分k8s的Pod并不是用户直接创建的。它们通常是作为[Deployment]()、[ReplicaSet]()或者其他模板化控制器的一部分。为控制授权策略访问，也就为这个控制器创建的*所有*Pod授权了策略访问，所以建议去给Pod的ServiceAccount做策略访问授权（见[栗子](#运行另一个Pod)）。

### 通过RBAC

[RBAC](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)是k8s的标准授权模型，对于策略授权也非常简单。

首先，一个`Role`或`ClusterRole`需要被授权`use`目标策略。授权规则类似这样：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role name>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <list of policies to authorize>
```

然后要把`(Cluster)Role`绑定到授权用户上：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <binding name>
roleRef:
  kind: ClusterRole
  name: <role name>
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize specific service accounts:
- kind: ServiceAccount
  name: <authorized service account name>
  namespace: <authorized pod namespace>
# Authorize specific users (not recommended):
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <authorized user name>
```

如果用的是`RoleBinding`（而不是`ClusterRoleBinding`），那就只能给和Binding在同一个命名空间中的Pod授权。可以配合系统组，为命名空间下的所有Pod进行授权：

```yaml
# Authorize all service accounts in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts
# Or equivalently, all authenticated users in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
```

关于RBAC绑定的栗子，见[角色绑定栗子](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/rbac#role-binding-examples)。关于PodSecurityPolicy授权的完整栗子，见[下面](#栗子)。

### 排错

- [Controller Manager](https://v1-18.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)必须运行在[安全的API端口](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/controlling-access/)，而且不能有超级用户权限。否则会导致请求会绕过认证和授权模型，所有的PodSecurityPolicy对象都会被允许，用户也可以创建特权容器。关于ControllerManager的授权配置，详见[Controller Role](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles)。

## 策略的顺序

除了可以限制Pod的创建和更新，Pod安全策略还可以为它所控制的字段提供默认值。当有多个可用的策略时，Pod安全策略控制器按照以下标准来选择策略：

- 1.优先使用不会改变Pod的PodSecurityPolicy。这种无修改策略的顺序也就无关紧要了。
- 2.如果Pod必须要有默认值或者要被修改，选择第一个可以允许Pod的PodSecurityPolicy（按名字排序）。

>**注意**：在更新的时候（此时无法改变Pod的定义），只会使用无修改的PodSecurityPolicy来校验Pod。

## 栗子

*这里我们假设你已经有了一个集群，并且开启了PodSecurityPolicy准入控制器，并且你是集群的管理员。*

### 安装

受限要创建一个命名空间和一个ServiceAccount来跑下面的栗子。我们用这个ServiceAccount来模拟一个非管理员用户。

```shell script
kubectl create namespace psp-example
kubectl create serviceaccount -n psp-example fake-user
kubectl create rolebinding -n psp-example fake-editor --clusterrole=edit --serviceaccount=psp-example:fake-user
```

为了明确我们用到哪个用户，减少一些重复的内容，我们建两个别名：

```shell script
alias kubectl-admin='kubectl -n psp-example'
alias kubectl-user='kubectl --as=system:serviceaccount:psp-example:fake-user -n psp-example'
```

### 创建一个策略和Pod

在文件中定义示例PodSecurityPolicy。这个策略会阻止特权Pod的创建。PodSecurityPolicy对象的名字必须是有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

通过kubectl来创建：

```shell script
kubectl-admin create -f example-psp.yaml
```

现在，作为非特权用户，创建一个普通的Pod：

```shell script
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pause" is forbidden: unable to validate against any pod security policy: []
```

**发生了什么？**尽管创建了PodSecurityPolicy，不论是Pod的ServiceAccount还是`fake-user`都没有权限使用这个新的策略：

```shell script
kubectl-user auth can-i use podsecuritypolicy/example
no
```

创建rolebinding，为`fake-user`授权策略的`use`动词：

>**注意**：这可不是推荐的方式！真正规范的用法看[下一节](#运行另一个Pod)。

```shell script
kubectl-admin create role psp:unprivileged \
    --verb=use \
    --resource=podsecuritypolicy \
    --resource-name=example
role "psp:unprivileged" created

kubectl-admin create rolebinding fake-user:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=psp-example:fake-user
rolebinding "fake-user:psp:unprivileged" created

kubectl-user auth can-i use podsecuritypolicy/example
yes
```

现在再来创建Pod：

```shell script
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      pause
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
EOF
pod "pause" created
```

有结果了！但如果要创建特权Pod的话依然会被拒绝：

```shell script
kubectl-user create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name:      privileged
spec:
  containers:
    - name:  pause
      image: k8s.gcr.io/pause
      securityContext:
        privileged: true
EOF
Error from server (Forbidden): error when creating "STDIN": pods "privileged" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]
```

继续下面的学习之前删除这个Pod：

```shell script
kubectl-user delete pod pause
```

### 运行另一个Pod

这次我们闹个不太一样的：

```text
kubectl-user create deployment pause --image=k8s.gcr.io/pause
deployment "pause" created

kubectl-user get pods
No resources found.

kubectl-user get events | head -n 2
LASTSEEN   FIRSTSEEN   COUNT     NAME              KIND         SUBOBJECT                TYPE      REASON                  SOURCE                                  MESSAGE
1m         2m          15        pause-7774d79b5   ReplicaSet                            Warning   FailedCreate            replicaset-controller                   Error creating: pods "pause-7774d79b5-" is forbidden: no providers available to validate pod request
```

**发生了什么？**我们已经把`psp:unprivileged`角色绑定给了`fake-user`，为什么报了个`Error creating: pods "pause-7774d79b5-" is forbidden: no providers available to validate pod request`？关键在于SOURCE列的`replicaset-controller`。我们的fake-user用户成功的创建了Deployment（它又成功的创建了一个ReplicaSet），但是当ReplicaSet要创建Pod的时候，它没有被授权使用栗子中的PodSecurityPolicy。

要想跑通，需要将`psp:unprivileged`绑定到Pod的ServiceAccount上。本例中（因为我们没有指定）ServiceAccount是`default`：

```shell script
kubectl-admin create rolebinding default:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=psp-example:default
rolebinding "default:psp:unprivileged" created
```

然后再重试协议爱，副本控制器应该就可以创建Pod了：

```text
kubectl-user get pods --watch
NAME                    READY     STATUS    RESTARTS   AGE
pause-7774d79b5-qrgcb   0/1       Pending   0         1s
pause-7774d79b5-qrgcb   0/1       Pending   0         1s
pause-7774d79b5-qrgcb   0/1       ContainerCreating   0         1s
pause-7774d79b5-qrgcb   1/1       Running   0         2s
```

### 清理一下

直接删除命名空间可以清理大部分示例资源：

```shell script
kubectl-admin delete ns psp-example
namespace "psp-example" deleted
```

`PodSecurityPolicy`不是基于命名空间的资源，需要单独删除：

```shell script
kubectl-admin delete psp example
podsecuritypolicy "example" deleted
```

### 示例策略

这是一个你能创建的最小限制的策略，基本上等于没有使用Pod安全策略的准入控制器：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

下面是一个受限的策略，需要非特权用户，阻止提权到root，用到了几个安全机制。

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # Required to prevent escalations to root.
  allowPrivilegeEscalation: false
  # This is redundant with non-root + disallow privilege escalation,
  # but we can provide it for defense in depth.
  requiredDropCapabilities:
    - ALL
  # Allow core volume types.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # Assume that persistentVolumes set up by the cluster admin are safe to use.
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # Require the container to run without root privileges.
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

[Pod安全标准](../安全/Pod安全标准.md#策略实例化)中有更多栗子。

## 策略参考

### Privileged

**Privileged**——决定容器是否可以启用特权模式。默认情况下容器是不允许访问主机上的任何设备的，但是“特权”容器就可以。它可以让容器几乎可以拥有和主机进程一样的权限。对于要使用Linux Capabilities的容器这就比较有用，比如操作网络栈以及访问设备。

### 主机命名空间

**HostPID**——控制容器是否可以共享主机的进程ID的namespace。注意一旦和ptrace配合使用的话，可以提权到容器外部（默认禁用ptrace）。

**HostIPC**——控制容器是否可以共享主机的IPC的namespace。

**HostNetwork**——控制Pod是否可以使用节点的网络namespace。这样可以允许Pod访问环回接口，作为localhost上的服务，可以嗅探到同一个节点上其他Pod的网络活动。

**HostPorts**——提供一个可以开放在主机网络namespace中的端口列表。用`HostPortRange`来定义，带有`min`（包含）和`max`（不包含）。默认不允许主机端口。

### 数据卷和文件系统

**Volumes**——提供一个允许的数据卷类型列表。这些允许的值对应的是创建数据卷的时候它们的来源。完整的数据卷类型见[数据卷类型](../存储/数据卷.md#数据卷类型)。可以用`*`来允许所有的数据卷类型。

对于新的PSP，**推荐最小化的集合**：

- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- secret
- projected

>**警告**：PodSecurityPolicy不会限制`PVC`引用的`PV`对象的类型数量，hostPath类型的`PV`也不支持只读访问模式。只有授信用户才应该有权限创建`PV`对象。

**FSGroup**——控制一些数据卷的补充用户组。

- *MustRunAs*——至少定义一个`range`。以第一个range的最小值为默认值。会校验所有range。
- *MayRunAs*——至少定义一个`range`。可以允许`FSGroups`留空不设默认值。如果设置了`FSGroups`，会校验所有range。
- *RunAsAny*——没有默认值。允许所有指定的`fsGroup`ID。

**AllowedHostPaths**——指定允许用在hostPath数据卷的主机路径。空列表的话就没限制。它的值是一个对象列表，每个对象有一个`pathPrefix`字段，允许hostPath数据卷挂载到相同前缀的路径上，`readOnly`字段表示要以只读方式挂载。比如：

```yaml
allowedHostPaths:
  # This allows "/foo", "/foo/", "/foo/bar" etc., but
  # disallows "/fool", "/etc/foo" etc.
  # "/foo/../" is never valid.
  - pathPrefix: "/foo"
    readOnly: true # only allow read-only mounts
```

>**警告**：
>
>对于不受限访问主机文件系统的容器，有很多方式可以实现提权，包括读取其他容器的数据，滥用系统服务的凭证，比如kubelet。
>
>可写的hostPath目录允许容器遍历到`pathPrefix`外部的主机文件系统。从1.11+开始，`readOnly: true`必须要用于**所有**`allowedHostPaths`上，有效的将访问限制在`pathPrefix`上。

**ReadOnlyRootFilesystem**——要求容器必须用只读的根文件系统（比如无可写层）。

### FlexVolume驱动

Flex数据卷只能使用列表中指定的FlexVolume驱动。空列表，或者是nil值，则代表没有限制。一定要确保[`volumes`](#数据卷和文件系统)字段中包含`flexVolume`数据卷类型；否则不会允许任何FlexVolume驱动。

比如：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: allow-flex-volumes
spec:
  # ... other spec fields
  volumes:
    - flexVolume
  allowedFlexVolumes:
    - driver: example/lvm
    - driver: example/cifs
```

### 用户和组

**RunAsUser**——控制容器运行使用的用户ID。

- *MustRunAs*——要求至少定义一个`range`。用第一个range中的最小值作为默认值。会校验所有range。
- *MustRunAsNonRoot*——要求Pod的`runAsUser`必须非零，或者在镜像中使用了`USER`指令（通过数字UID指定）。如果Pod既没有`runAsNonRoot`也没有`runAsUser`，那就会自动被修改成`runAsNonRoot=true`，因此需要在容器中定义一个非零数字型的`USER`指令。这里不会有默认值。这种场景下强烈建议设置`allowPrivilegeEscalation=false`。
- *RunAsAny*——没有默认值。允许定义任意的`runAsUser`。

**RunAsGroup**——控制容器运行的主用户组ID。

- *MustRunAs*——要求至少定义一个`range`。用第一个range中的最小值作为默认值。会校验所有range。
- *MayRunAs*——不需要定义RunAsGroup。但是一旦有了RunAsGroup，那就必须要存在云已定义的range中。
- *RunAsAny*——没有默认值。允许定义任意的`runAsGroup`。

**SupplementalGroups**——控制容器添加的用户组ID。

- *MustRunAs*——要求至少定义一个`range`。用第一个range中的最小值作为默认值。会校验所有range。
- *MayRunAs*——至少定义一个`range`。允许没有`supplementalGroups`。如果有，会校验所有range。
- *RunAsAny*——没有默认值。允许定义任意的`supplementalGroups`。

### 提权

这个用来控制容器的`allowPrivilegeEscalation`选项。这个布尔值直接控制着是否会给容器进程设置[`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt)选项。这个选项会阻止用`setuid`来修改生效的用户ID，保护文件以免被开启额外的cap（比如会阻止使用`ping`工具）。这个策略可以用于有效地实施`MustRunAsNonRoot`。

**AllowPrivilegeEscalation**——控制是否允许用户将容器的SecurityContext设置为`allowPrivilegeEscalation=true`。它会默认允许，不会阻止setuid。将它设置为`false`就可以保证容器的子进程不会得到比父进程更多的权限。

**DefaultAllowPrivilegeEscalation**——设置`allowPrivilegeEscalation`的默认值。如果没有这个策略的辅助，默认就是允许提权，不会阻止setuid。如果这种默认行为不是你想要的，那这个字段就可以用来讲默认行为改为拒绝，同时还能允许Pod主动声明`allowPrivilegeEscalation`。

### Capabilities

Linux的capabilities是对传统超级用户权限更细粒度的分解。其中一些capabilities可以用来提权或者让容器越狱，可能会被PodSecurityPolicy限制。关于Linux的capabilities，见[capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)。

下面的字段都包含了一个capabilities列表，定义方式同ALL_CAPS中的capability名字，无`CAP_`前缀。

**AllowedCapabilities**——包含了可以为容器添加的capabilities的列表。这里的默认集合是隐式允许的。如果集合为空则意味着除了默认的集合外不允许添加其他的capabilities。可以用`*`来允许所有的capabilities。

**RequiredDropCapabilities**——这里的capabilities必须从容器中删除。这些capabilities会从默认集合中删除，绝对不能再添加。`RequiredDropCapabilities`中的capabilities不能出现在`AllowedCapabilities`或`DefaultAddCapabilities`中。

**DefaultAddCapabilities**——默认添加给容器的capabilities，还要加上运行时提供的默认值。如果运行时是Docker，去看一下[Docker文档](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)中关于默认的capabilities有哪些。

### SELinux

- **MustRunAs**——需要配置`seLinuxOptions`。默认使用`seLinuxOptions`。校验所有的`seLinuxOptions`。
- **RunAsAny**——没有默认值。允许任意的`seLinuxOptions`。

### AllowedProcMountTypes

`allowedProcMountTypes`是一个可用ProcMountType列表。为空或者为nil意味着只能用`DefaultProcMountType`。

`DefaultProcMount`用容器运行时的默认值为/proc做只读处理或者屏蔽某些路径。大部分容器运行时都会屏蔽/proc下的某些路径，避免将特殊的设备或者信息暴露出来导致安全隐患。具体设置方式为字符串`Default`。

其他的ProcMountType只有一种，那就是`UnmaskedProcMount`，它会绕过容器运行时的默认屏蔽行为，保证容器新创建的/proc能够保持原样不被修改。具体设置方式为字符串`Unmasked`。

### AppArmor

通过PodSecurityPolicy上面的注解来控制。参见[AppArmor文档](https://v1-18.docs.kubernetes.io/docs/tutorials/clusters/apparmor/#podsecuritypolicy-annotations)。

### Seccomp

Pod中关于seccomp的配置可以通过PodSecurityPolicy上面的注解来控制。seccomp目前在k8s中还是一个alpha版本的功能。

**seccomp.security.alpha.kubernetes.io/defaultProfileName**——用来设置为容器应用的默认seccomp配置注解。可能的值包括：

- `unconfined`——如果没有提供其他选择，不会为容器进程应用seccomp（这是k8s中的默认行为）。
- `runtime/default`——使用磨人的容器运行时配置。
- `docker/default`——使用Docker默认的seccomp配置。从1.11开始已经弃用了。用`runtime/default`代替。
- `localhost/<path>`——在节点的`<seccomp_root>/<path>`路径下定义一个配置文件，`<seccomp_root>`是通过kubelet的`--seccomp-profile-root`选项来定义的。

**seccomp.security.alpha.kubernetes.io/allowedProfileNames**——这个注解用来定义允许哪些值可以用在Pod的seccomp注解上。具体值为逗号间隔的可用值。可能的值上面已经列出来了，再加上`*`，代表允许所有配置。如果没有这个注解，意味着不能修改默认值。

### Sysctl

默认是允许所有安全的sysctl的。

- `forbiddenSysctls`——排除所有指定的sysctl。可以在这个列表中禁用安全和不安全的sysctl。要想禁用全部，那就设置`*`。
- `allowedUnsafeSysctls`——对于默认禁用的sysctl可以在这里启用，但是它们不能同时出现在`forbiddenSysctls`中。

## 下一步……

- 学习[Pod安全标准](../安全/Pod安全标准.md)了解推荐的策略。
- 参考[Pod安全策略参考手册](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podsecuritypolicy-v1beta1-policy)了解API的细节。