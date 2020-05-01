# Pod预设值

本文介绍PodPresets，它可以用来在Pod创建的时候注入一些信息。这些信息包括secret、数据卷、卷挂载以及环境变量。

- [理解Pod预设值](#理解Pod预设值)
- [它是咋工作的](#它是咋工作的)
- [启用Pod预设值](#启用Pod预设值)
- [接下来……](#接下来)

## 理解Pod预设值

`Pod Preset`也是一种API资源，用来在Pod创建的时候为其注入一些额外的运行时需求。使用[标签选择器](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)为PodPreset指定对应的Pod。

使用PodPreset，可以让用户不必为每个Pod都定义全部的信息。此时，Pod模板的编写者在使用一个服务时，就不需要知道这个服务的全部细节了。

关于这玩意的来由，可以看一下它的[设计思想](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/service-catalog/pod-preset.md)。

## 它是咋工作的

k8s中的admission controller可以开启`PodPreset`，然后，后续的Pod创建就可以注入预设值了。当创建Pod时，系统会进入以下流程：

- 1.获取所有可用的`PodPresets`。
- 2.检查是否存在跟Pod标签匹配的`PodPreset`。
- 3.尝试将`PodPreset`中定义的各种资源合并到正在创建的Pod中。
- 4.如果出错，Pod中会显示错误信息，*忽略*所有`PodPreset`中定义的资源，直接创建Pod。
- 5.给Pod中增加注解，标识出这是一个被`PodPreset`修改过的Pod。格式如`podpreset.admission.kubernetes.io/podpreset-<pod-preset name>: "<resource version>"`

每个Pod可以匹配到多个Pod预设值；每个`PodPreset`也可以应用到多个Pod上。当`PodPreset`生效之后，k8s会修改Pod的Spec。如果修改了`Env`、`EnvFrom`和`VolumeMounts`，k8s会修改Pod中所有容器的定义；如果修改了`Volume`，k8s会修改Pod的定义。

>**注意**：Pod预设值还可以用来修改以下字段：`.spec.containers`、`initContainers`字段（后者需要k8s版本大于等于1.14.0），记住了吗，小朋友们。

### 关闭某个Pod的预设值

可能有些Pod你不想让它们被预设值糟蹋了。此时可以在Pod定义中增加一个注解：`podpreset.admission.kubernetes.io/exclude: "true"`。

## 启用Pod预设值

如果要开启Pod预设功能，需要以下操作：

- 1.开启API`settings.k8s.io/v1alpha1/podpreset`。将`settings.k8s.io/v1alpha1=true`添加到apiserver的启动选项`--runtime-config`中。如果用的是minikube，启动集群时添加`--extra-config=apiserver.runtime-config=settings.k8s.io/v1alpha1=true`。
- 2.开启`PodPreset`的admission controller。将`PodPreset`添加到apiserver的`--enable-admission-plugins`选项中。如果用的时minikube，需要如下设置：

```text
--extra-config=apiserver.enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset
```

- 3.创建好你要用的`PodPreset`对象。

## 接下来……

- [实操]()