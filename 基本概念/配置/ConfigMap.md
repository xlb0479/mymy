# ConfigMap

ConfigMap也是一个API对象，用KV形式保存非敏感信息。[Pod](../业务组件/泡德（Pod）/Pod.md)可以将ConfigMap用作环境变量、命令行参数，或者是保存在[数据卷](../存储/数据卷.md)中的配置文件。

ConfigMap可以将环境相关的配置信息从[容器镜像](https://kubernetes.io/docs/reference/glossary/?all=true#term-image)中分离出去，这样可以让你的应用可移植性更好。

>**小心**：ConfigMap不提供加解密功能。如果要存敏感信息，那就用[Secret](Secret.md)而不是Configmap，或者用其他（第三方）工具保证数据的私有性。

## 动机

用ConfigMap将配置信息从应用代码中分离出来。

比如你正在开发一个可以运行在你自己电脑上的程序（开发环境），还可以运行在云上（接收真实的流量）。你写了一段代码，在环境变量中查找一个名为`DATABASE_HOST`的变量。本地开发的时候，你将这个变量设置成了`localhost`。云上的时候，你把它指向了一个k8s的[Service](../Service，负载均衡，网络/Service.md)，这个Service将数据库暴露到了你的集群中。

这样就可以在云上拉取一个容器镜像，跟你本地代码进行一模一样的调试。

## ConfigMap对象

ConfigMap是一个API[对象](../概要/Kubernetes对象/理解Kubernetes对象%20.md)，可以保存用来为其他对象提供的配置信息。和其他k8s对象不太一样，其他对象有一个`spec`，而ConfigMap是用一个`data`来保存数据的KV格式。

ConfigMap的名字必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

## ConfigMap和Pod

可以在Pod的`spec`中引用一个ConfigMap，用ConfigMap中保存的数据来配置Pod中的容器（们）。Pod和ConfigMap必须在相同的[命名空间](../概要/Kubernetes对象/命名空间.md)中。

下面是一个ConfigMap的栗子，其中一些key值是单一的，另一些key的值看上去应该是某种配置格式的一部分。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  #
  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

有四种方式可以用来让ConfigMap对Pod中的容器进行配置：

- 1.为容器的entrypoint添加的命令行参数
- 2.为容器提供环境变量
- 3.将文件放在只读数据卷中，提供给应用读取
- 4.在Pod中通过编写代码调用k8s的API来读取ConfigMap

这些不同的方法也为配置信息提供了不同的建模方式。对于前三种，[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)在启动容器的时候就可以读取ConfigMap中的数据。

第四种方式就是说你要写一些代码来读取ConfigMap和它的数据。但由于你直接使用了k8s的API，应用可以订阅并在ConfigMap更新的时候感知到这些变化，并做出相应的反应。通过直接调用k8s的API，这种方式可以让你访问到其他命名空间中的ConfigMap。

下面的栗子是一个Pod用`game-demo`中的数据来配置Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: game.example/demo-game
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

ConfigMap不会区分你是单行属性还是多行属性。重要的是Pod和其他对象要怎么使用这些属性。

对于上面的栗子，定义了一个数据卷并作为`/config`挂载到`demo`容器中，它里面创建了两个文件，`/config/game.properties`和`/config/user-interface.properties`，尽管ConfigMap中一共有四个key。这是因为Pod定义中的`volumes`部分指定了一个`items`的数组。如果你直接忽略掉`items`数组，ConfigMap中的每一个key都会变成一个文件，文件名就是key，你就得到了4个文件。

## 使用ConfigMap

ConfigMap可以当做是数据卷来挂载。ConfigMap也可以被用于系统的其他部分，而不是直接暴露到Pod中。比如ConfigMap可以保存系统中其他部分用来做配置的数据。

>**注意**：
>ConfigMap最常见的用法就是用来配置同一个命名空间下的Pod中的容器。当然你也可以单独使用一个ConfigMap。
>
>比如你可能见过一些[插件](https://kubernetes.io/docs/concepts/cluster-administration/addons/)或[oeprator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)，基于一个ConfigMap来调整它们的具体行为。

### 将ConfigMap用作Pod中的文件

要想在Pod中通过数据卷的方式来使用ConfigMap：

- 1.创建或者用已有的ConfigMap。多个Pod可以引用同一个ConfigMap。
- 2.修改Pod定义，在`.spec.volumes[]`中添加一个数据卷。名字随便起，设置一个`.spec.volumes[].configMap.name`字段，引用你的ConfigMap对象。
- 3.为每个需要ConfigMap的容器中添加一个`.spec.containers[].volumeMounts[]`。设置`.spec.containers[].volumeMounts[].readOnly = true`，将`.spec.containers[].volumeMounts[].mountPath`设置到一个未使用的目录，ConfigMap就会悄然出现在里面。
- 4.修改镜像或命令行参数，让程序能够在那个目录中读取文件。ConfigMap的`data`中的每个key都会变成`mountPath`下的一个文件名。

下面的栗子就是一个Pod将一个ConfigMap弄成了一个数据卷挂载了：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

你用到的ConfigMap都要添加到`.spec.volumes`中。

如果Pod中有多个容器，每个容器要设置自己的`volumeMounts`，但每个ConfigMap只需要一个`.spec.volumes`。

#### 挂载自动更新的ConfigMap

当一个正在使用的ConfigMap被更新了，对应的key最终也会被更新。kubelet在每次定时同步的时候都要检查被挂载的ConfigMap是否保持最新。但是kubelet是用了本地缓存来获取ConfigMap的当前值的。缓存的类型可以通过[KubeletConfiguration结构体](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go)中的`ConfigMapAndSecretChangeDetectionStrategy`来进行配置。一个ConfigMap可以通过监视（默认）、ttl或者直接将所有请求重定向到apiserver的方式进行传播。结果就是，从ConfigMap被更新开始，到新的key被映射到Pod中那一刻，最长的时间延迟就是kubelet的同步周期+缓存传播延迟，而缓存传播延迟依赖于选择的缓存类型（它等于监视传播延时，缓存ttl，或者两者都是零）。

**功能状态**：`Kubernetes v1.18 [alpha]`

k8s的alpha阶段特性，*不可变Secret和ConfigMap*，可以将单独的Secret和ConfigMap设置为不可变的。对于大范围使用ConfigMap的集群（上万个ConfigMap独立挂载到Pod中），阻止数据修改可以提供以下好处：

- 避免你不小心（或有害的）更新导致应用出错
- 极大的降低了apiserver的负载，提升了集群的性能，因为不可变的ConfigMap就不需要一直被监视着了。

要使用这个功能，需要开启`ImmutableEphemeralVolumes`[特性门](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)，将Secret或ConfigMap的`immutable`字段设置为`true`。比如：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

>**注意**：一旦ConfigMap或Secret被标记为不可修改的，这个操作*无法*回退，而且也不能修改`data`字段的值。你只能删除或重建这个ConfigMap。之前的Pod会一直保留指向被删除的ConfigMap的挂载点——建议对这些Pod也进行重建。

## 下一步……

- 阅读[Secret](Secret.md)。
- 阅读[在Pod中配使用ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)。
- 阅读[The Twelve-Factor App](https://12factor.net/)，理解为什么要将配置信息从应用代码中分离出来。