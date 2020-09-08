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

### 运行另一个Pod

## 策略参考

### Privileged

### 主机命名空间

### 数据卷和文件系统

### FlexVolume驱动

### 用户和组

### 提权

### Capabilities

### SELinux

### AllowedProcMountTypes

### AppArmor

### Seccomp

### Sysctl