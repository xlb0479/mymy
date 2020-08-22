# Secret

使用Secret可以用来保存和管理一些敏感信息，比如密码、OAuth的token、ssh的秘钥等等。把敏感信息保存到Secret中要比直接放到[Pod](../业务组件/泡德（Pod）/Pod.md)定义或者[容器镜像](https://kubernetes.io/docs/reference/glossary/?all=true#term-image)中更加的灵活和安全。详见[Secret设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/auth/secrets.md)。

## 概要

Secret是用来保存少数敏感信息的对象，比如密码、token、key。这种信息之前可能是直接放在Pod定义或者镜像里面。用户可以自己创建一些Secret，同样，系统本身也创建了一些Secret。

使用Secret必须要在Pod中引用这个Secret。Pod可以通过三种方式使用Secret：

- 作为[数据卷](../存储/数据卷.md)中的[文件](#Secret作为Pod中的文件)，挂载到一个或多个容器中。
- 作为[容器的环境变量](#Secret作为环境变量)。
- [给kubelet为Pod拉镜像用](#用作imagePullSecrets)。

### 内置Secret

#### ServcieAccount自动创建Secret并添加API凭据

k8s会自动创建Secret，其中包含访问API的凭据，自动修改你的Pod，让它可以使用这个Secret。

这种自动创建和使用API凭据的方式可以被关掉或者覆盖。但如果你只是想安全的访问apiserver，那就应该用这种推荐的方式。

详见[ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)。

### 创建Secret

#### 用`kubectl`创建一个Secret

Secret可以用来保存Pod访问数据库时需要的用户凭据。比如一个包含了用户名和密码的数据库链接字符串。可以先在你的本地将用户名保存到`./username.txt`文件，将密码保存到`./password.txt`文件。

```shell script
# Create files needed for the rest of the example.
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt
```

`kubectl create secret`命令可以将这些文件打包到一个Secret中，并在apiserver上创建相应的对象。Secret对象的名字必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。

```shell script
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
```

执行后有如下输出：

```shell script
secret "db-user-pass" created
```

默认key名就是文件名。你也可以通过`[--from-file=[key=]source]`格式来设置key名。

```shell script
kubectl create secret generic db-user-pass --from-file=username=./username.txt --from-file=password=./password.txt
```

>**注意**：
>
>类似`$`、`\`、`*`、`=`以及`!`这种特殊字符会在你的[shell](https://en.wikipedia.org/wiki/Shell_(computing))中被解释，所以需要转义。在大部分的shell中，密码转义的最简单的方法就是加上单引号（`'`）。比如你的密码是`S!B\*d$zDsb=`，就要像这样执行命令：
>
>```shell script
>kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
>```
>
>如果是保存在文件中的密码就不用转义了（`--from-file`）。

检查一下创建好的Secret：

```shell script
kubectl get secrets
```

输出如下：

```text
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s
```

查看Secret的描述：

```shell script
kubectl describe secrets/db-user-pass
```

输出如下：

```text
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

>**注意**：`kubectl get`和`kubectl describe`命令默认不会显示Secret中的内容。这是为了避免Secret暴露给不经意的观察者，或者被保存到终端的日志中。

阅读[Secret解码](#Secret解码)，了解如何查看Secret中的内容。

#### 手动创建Secret

可以先在文件中创建一个Secret，JSON或者YAML格式的，然后再创建具体的对象。Secret对象的名字必须是一个有效的[DNS子域名](../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。[Secret](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#secret-v1-core)包含了两个map：`data`和`stringData`。`data`字段用来保存任意的数据，用base64编码。`stringData`字段是为了方便一些，可以直接保存未编码的敏感数据。

比如要在Secret中的`data`字段保存两个字符串，先将字符串做base64编码：

```shell script
echo -n 'admin' | base64
```

输出如下：

```text
YWRtaW4=
```

```shell script
echo -n '1f2d1e2e67df' | base64
```

输出如下：

```text
MWYyZDFlMmU2N2Rm
```

然后Secret就长这样：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

用[`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply)来创建这个Secret对象吧：

```shell script
kubectl apply -f ./secret.yaml
```

输出如下：

```text
secret "mysecret" created
```

某些情况你可能需要用到`stringData`字段。这个字段可以让你将非base64编码的字符串直接保存到Secret中，Secret在创建或者更新的时候会自动进行编码。

这种情况的一个具体栗子就是，当你用的应用使用Secret来保存配置文件，你希望配置文件的部分内容在部署的过程中进行注入。

比如，你的应用要使用这样的配置文件：

```yaml
apiUrl: "https://my.api.com/api/v1"
username: "user"
password: "password"
```

可以把它保存到一个Secret中：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  config.yaml: |-
    apiUrl: "https://my.api.com/api/v1"
    username: {{username}}
    password: {{password}}
```

你的部署工具可以在`kubectl apply`之前替换掉`{{username}}`和`{{password}}`这两个模板变量。

`stringData`字段是一个只写（write-only）字段。获取Secret的时候你是看不到这个字段的。比如运行下面的命令：

```shell script
kubectl get secret mysecret -o yaml
```

输出如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:40:59Z
  name: mysecret
  namespace: default
  resourceVersion: "7225"
  uid: c280ad2e-e916-11e8-98f2-025000000001
type: Opaque
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19
```

如果一个字段，比如`username`，在`data`和`stringData`中都出现了，那就使用`stringData`中的值。比如下面这种：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
stringData:
  username: administrator
```

生成的Secret就是：

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:46:46Z
  name: mysecret
  namespace: default
  resourceVersion: "7579"
  uid: 91460ecb-e917-11e8-98f2-025000000001
type: Opaque
data:
  username: YWRtaW5pc3RyYXRvcg==
```

`YWRtaW5pc3RyYXRvcg==`解码之后的结果是`administrator`。

`data`和`stringData`的key必须是字母数字以及“-”、“_”或“.”。

>**注意**：Secret的数据在JSON或者YAML序列化之后会编码成base64字符串。在这些字符串中的换行是无效的，必须去掉。如果是在Darwin/macOS上面用的`base64`工具，用户要避免使用`-b`选项导致结果被分成多行。相反，Linux用户在使用`base64`命令的时候*应当*添加`-w 0`，或者如果`-w`选项不可用的话那就用管道`base64 | tr -d '\n'`进行处理。

#### 从生成器中创建一个Secret

从1.14版本开始，`kubectl`支持[用Kustomize管理对象](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)。Kustomize提供了资源生成器（Generator）来创建Secret和ConfigMap。Kustomize生成器应当定义在某个目录下的`kustomization.yaml`文件中。生成Secret之后就可以通过`kubectl apply`在apiserver上面创建Secret了。

#### 从文件中生成一个Secret

可以定义一个`secretGenerator`，从./username.txt和./password文件中生成一个Secret：

```shell script
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: db-user-pass
  files:
  - username.txt
  - password.txt
EOF
```

应用这个包含`kustomization.yaml`文件的目录来创建Secret：

```shell script
kubectl apply -k .
```

输出如下：

```shell script
secret/db-user-pass-96mffmfh4k created
```

看一下创建好的Secret：

```shell script
kubectl get secrets
```

输出如下：

```text
NAME                             TYPE                                  DATA      AGE
db-user-pass-96mffmfh4k          Opaque                                2         51s
```

```shell script
kubectl describe secrets/db-user-pass-96mffmfh4k
```

输出如下：

```text
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

#### 用普通字符串来生成Secret

可以定义一个`secretGenerator`，用普通的字符串`username=admin`和`password=secret`来创建一个Secret：

```shell script
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: db-user-pass
  literals:
  - username=admin
  - password=secret
EOF
```

应用这个包含`kustomization.yaml`文件的目录来创建Secret：

```shell script
kubectl apply -k .
```

输出如下：

```text
secret/db-user-pass-dddghtt9b5 created
```

>**注意**：一个Secret在生成后，它的名字是在name后面加上对Secret数据做哈希后的结果值。这就保证了每次数据被修改之后都会生成一个新的Secret。

#### Secret解码

可以通过`kubectl get secret`来获取Secret。比如你可以获取一下上面已经创建好了的Secret：

```shell script
kubectl get secret mysecret -o yaml
```

输出如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

对`password`字段进行解码：

```shell script
echo 'MWYyZDFlMmU2N2Rm' | base64 --decode
```

输出如下：

```text
1f2d1e2e67df
```

#### 编辑Secret

可以用下面的命令来编辑一个Secret：

```shell script
kubectl edit secrets mysecret
```

此时会打开默认的配置编辑器，可以更新在`data`字段中保存的base64编码后的数据：

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: { ... }
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```

## 使用Secret

Secret可以作为数据卷进行挂载，或者暴露到容器的[环境变量](../容器/容器环境.md)中。Secret还可以被系统的其他部分使用，不用直接暴露到Pod里。比如你可以让Secret来保存系统中的某些组件跟外部系统交互时用到的凭据信息。

### Secret作为Pod中的文件

要在Pod中从数据卷里获取一个Secret：

- 1.创建或使用一个已经存在的Secret。多个Pod可以引用同一个Secret。
- 2.修改Pod定义，在`.spec.volumes[]`中添加一个数据卷。名字随便起，但是`.spec.volumes[].secret.secretName`字段要跟Secret对象的名字保持一致。
- 3.给每个需要用到Secret的容器添加一个`.spec.containers[].volumeMounts[]`。设置`.spec.containers[].volumeMounts[].readOnly = true`，并将`.spec.containers[].volumeMounts[].mountPath`设置成一个未被使用的目录，Secret就会悄然出现在里面。
- 4.修改镜像或者命令行，让程序去这个目录下找文件去。Secret的`data`中的每个key会变成`mountPath`下的一个文件名。

下面是一个将Secret用作数据卷文件的Pod：

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
    secret:
      secretName: mysecret
```

你要用到的Secret都要加到`.spec.volumes`中。

如果Pod中有多个容器，每个容器要定义自己的`volumeMounts`，但是`.spec.volumes`每个Secret只需要添加一次。

可以将多个文件打包到一个或者多个Secret中，看你情况了。

#### 将Secret的key映射到指定路径上

你还可以控制Secret的key在数据卷中的映射路径。可以使用`.spec.volumes[].secret.items`为每个key修改目标路径：

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
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

结果就是：

- `username`数据会保存到`/etc/foo/my-group/my-username`而非`/etc/foo/username`。
- `password`数据则没有被映射。

一旦你用了`.spec.volumes[].secret.items`，只有`items`中指定的key会被映射。要想使用Secret中的所有key，那么所有key都要放到`items`字段里。反过来也一样，`items`字段中的所有key也必须要存在于对应的Secret中。否则数据卷就不会被创建。

#### Secret文件权限

可以为每个Secret的key设置文件访问权限位。如果不设置，默认就是`0644`。如果需要的话，可以为整个Secret数据卷设置一个默认的模式。

比如这样：

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
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 0400
```

Secret会被挂载到`/etc/foo`中，创建的文件权限都是`0400`。

注意JSON格式下是不支持八进制标记的，所以要用256而不是0400。如果你用的是YAML而不是JSON，那你就可以用八进制，这样的你看着更像个人。

如果你通过`kubectl exec`钻到了Pod里，你得追一下链接标记才能找到最终的文件模式。比如这样：

```shell script
kubectl exec mypod -it sh

cd /etc/foo
ls -l
```

输出如下：

```text
total 0
lrwxrwxrwx 1 root root 15 May 18 00:18 password -> ..data/password
lrwxrwxrwx 1 root root 15 May 18 00:18 username -> ..data/username
```

追着链接找到正确的文件模式。

```shell script
cd /etc/foo/..data
ls -l
```

输出如下：

```text
total 8
-r-------- 1 root root 12 May 18 00:18 password
-r-------- 1 root root  5 May 18 00:18 username
```

可以参照前面的栗子用映射的方式给每个文件设置不同的权限：

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
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
        mode: 0777
```

最终，`/etc/foo/my-group/my-username`的文件权限就是`0777`。如果用JSON，考虑到JSON的限制，你就要写成`511`。

注意这个权限值，过后再看的话可能会显示成小数格式。

#### 从数据卷中获取Secret的数据

在挂载了Secret数据卷的容器中，Secret的key作为文件名出现，Secret的值则是用base64编码保存到了文件中。执行对应的命令就是这样的结果：

```shell script
ls /etc/foo/
```

输出如下：

```text
username
password
```

```shell script
cat /etc/foo/username
```

输出如下：

```text
admin
```

```shell script
cat /etc/foo/password
```

输出如下：

```text
1f2d1e2e67df
```

容器中的程序要读取文件中的Secret数据。（这是什么意思呢？这不是明摆着呢么。奥。。。可能是。。。。。）

#### 自动更新

当一个作为数据卷的Secret被更新了，映射的key最终也会被更新。kubelet会在每个同步周期去检查挂载的Secret是不是最新的。但是kubelet会使用它的本地缓存来获取Secret的当前值。这种缓存的类型是通过[KubeletConfiguration结构体](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go)中的`ConfigMapAndSecretChangeDetectionStrategy`字段来控制的。Secret可以通过监视（默认）、ttl、或者直接将请求重定向到apiserver的方式进行传播。结果就是，从Secret被更新开始，到新的key被映射到Pod中，最大的延时时间是kubelet的同步周期+缓存传播延时，而缓存的传播延时取决于使用的缓存类型（它等于监视传播延时、缓存的ttl，或者两者皆为0）。

>**注意**：将Secret用作[subPath](../存储/数据卷.md#使用subPath)数据卷的话就无法感知到Secret的更新了。

**功能状态**：`Kubernetes v1.18 [alpha]`

k8s提供的alpha版本的特性*不可变Secret和ConfigMap*，可以将个别的Secret和ConfigMap设置为不可变的。对于大规模使用Secret的集群（成千上万个独立挂载的Secret），不让它们更新可以带来以下好处：

- 避免偶然的（不希望的）更新，导致应用异常
- 提升集群性能，极大降低了apiserver的负载，因为不可变的Secret就不需要对它们进行监视。

要使用这个功能，需要开启`ImmutableEphemeralVolumes`[特性门](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)，然后将Secret或ConfigMap的`immutable`字段设置为`true`。比如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true
```

>**注意**：一旦Secret或ConfigMap被设置为不可变的，这种操作是*不能*撤销的，而且也不能修改`data`字段的值。只能删除并重建Secret。Pod中会保留被删除的Secret的挂载点——建议同时对Pod进行重建。

### Secret作为环境变量

要想在Pod中通过[环境变量](../容器/容器环境.md)来使用Secret：

- 1.创建或使用一个已有的Secret。多个Pod可以引用同一个Secret。
- 2.修改相关的Pod定义，为Secret中的key添加一个环境变量。环境变量要通过`env[].valueFrom.secretKeyRef`来使用Secret中的key。
- 3.修改镜像或命令行，让程序能够看到这些环境变量。

一个栗子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

#### 从环境变量中使用Secret数据

在容器中，Secret的key作为普通的环境变量出现，值则是base64编码后的数据。下面的命令是钻到上面创建好的Pod中执行的：

```shell script
echo $SECRET_USERNAME
```

输出：

```text
admin
```

```shell script
echo $SECRET_PASSWORD
```

输出：

```text
1f2d1e2e67df
```

### 用作imagePullSecrets

`imagePullSecrets`字段是引用同一个命名空间下的Secret列表。可以使用`imagePullSecrets`将包含了Docker（或其他）镜像仓库密码的Secret传递给kubelet。kubelet会根据Pod的设置，使用这些信息来拉取私有仓库的镜像。关于`imagePullSecrets`的更多信息见[PodSpec API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podspec-v1-core)。

#### 手动设置一个imagePullSecret

可以从[容器镜像文档](../容器/镜像.md)中学习如果设置`imagePullSecrets`。

### 自动附加imagePullSecrets

可以手动创建`imagePullSecrets`，然后通过一个ServiceAccount来引用它。所有通过这个ServiceAccount创建的Pod，或者默认通过它进行创建，都会得到它里面设置的`imagePullSecrets`。关于这个小知识的详细介绍见[给ServiceAccount添加imagePullSecrets](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)。

### 为手动创建的Secret做自动挂载

手动创建的Secret（比如包含一个访问GitHub账户的token）可以基于ServiceAccount自动附加到Pod上。详见[使用PodPreset为Pod注入信息](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)。

## 详细内容

### 限制

Secret数据卷会进行校验，确保引用的对象真的是一个Secret对象。所以，Secret要在Pod之前进行创建。

Secret资源也是依赖于[命名空间](../概要/Kubernetes对象/命名空间.md)的。只能被同一个命名空间下的Pod引用。

每个Secret大小限制为1MB。这是为了避免创建过大的Secret，消耗apiserver和kubelet的内存。但是创建大量小Secret也同样消耗内存。对于Secret施加更复杂的内存限制是下一步的开发计划。

kubelet只支持通过apiserver来获取Secret然后再提供给Pod使用。这包括直接用`kubectl`创建的Pod，以及间接地通过副本控制器创建的Pod。不包括那些基于kubelet的`--manifest-url`、`--config`选项，还有REST API（一般不用这个创建Pod）创建的Pod。

如果Pod将Secret用作环境变量，Secret必须要在Pod之前创建，除非它们被标记成可选的。如果引用了不存在的Secret，Pod起不来。

如果引用（`secretKeyRef`字段）了Secret中不存在的key，Pod同样起不来。

如果通过`envFrom`导入Secret作为环境变量，而Secret中的某些key是无效的环境变量名，这些key就会被跳过。Pod可以正常启动。在事件列表中会有一条记录，原因为`InvalidVariableNames`，消息中包含了被跳过的无效的key列表。下面的栗子中的Pod引用了default/mysecret中的两个无效的key：`1badkey`和`2alsobad`。

```shell script
kubectl get events
```

输出如下：

```text
LASTSEEN   FIRSTSEEN   COUNT     NAME            KIND      SUBOBJECT                         TYPE      REASON
0s         0s          1         dapi-test-pod   Pod                                         Warning   InvalidEnvironmentVariableNames   kubelet, 127.0.0.1      Keys [1badkey, 2alsobad] from the EnvFrom secret default/mysecret were skipped since they are considered invalid environment variable names.
```

### Secret与Pod的生命周期

通过调用k8s的API创建了一个Pod后，不会检查它引用的Secret是否存在。当Pod被调度时，kubelet会尝试获取Secret的数据。如果Secret不存在，或者无法连接到apiserver，拉取不到Secret的数据，kubelet会重复尝试。它会报告一个相关的事件，告诉你为什么Pod还没起来。一旦拉到了Secret，kubelet就会创建一个包含它的数据卷然后进行挂载。直到Pod的所有数据卷挂在完成才会启动Pod。

## 用例

### 用例：作为容器环境变量

创建一个Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=
  PASSWORD: MWYyZDFlMmU2N2Rm
```

执行创建命令：

```shell script
kubectl apply -f mysecret.yaml
```

使用`envFrom`将Secret的所有数据用作环境变量。Secret的key就会变成环境变量的名字。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: mysecret
  restartPolicy: Never
```

### 用例：Pod和sshkey

创建一个包含ssh key的Secret：

```shell script
kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/path/to/.ssh/id_rsa --from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub
```

输出如下：

```text
secret "ssh-key-secret" created
```

还可以创建一个`kustomization.yaml`文件，用`secretGenerator`字段包含ssh的key。

>**小心**：使用ssh key之前你最好脑子机密点：集群的其他用户可能也能访问到这个Secret。为集群中的目标用户群体创建一个ServiceAccount，如果用户受到打压，被按在墙角逼他说出密码，那就可以撤销这个账户。

现在可以创建一个引用这个Secret的Pod，通过数据卷的形式来使用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
  labels:
    name: secret-test
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: ssh-key-secret
  containers:
  - name: ssh-test-container
    image: mySshImage
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```

容器跑起来之后，可以在下面的位置找到key：

```shell script
/etc/secret-volume/ssh-publickey
/etc/secret-volume/ssh-privatekey
```

容器可以自由使用这些key来建立ssh连接。

### 用例：持有生产/测试凭据的Pod

本例中有俩Pod，一个Pod用到了包含生产环境凭据的Secret，另一个Pod用到了包含测试环境凭据的Secret。

可以用包含`secretGenerator`字段的`kustomization.yaml`文件，或者直接用`kubectl create secret`命令来创建。

```shell script
kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11
```

输出如下：

```text
secret "prod-db-secret" created
```

```shell script
kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests
```

输出如下：

```text
secret "test-db-secret" created
```

>**注意**：
>
>诸如`$`、`\`、`*`、`=`以及`!`这种特殊字符会被[shell](https://en.wikipedia.org/wiki/Shell_(computing))进行解释，所以需要转义。在大部分shell中，最简单的转义方式就是加单引号（`'`）。比如你的密码是`S!B\*d$zDsb=`，那你的命令就应该是这样：
>
>```shell script
>kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
>```
>
>如果是基于文件进行创建（`--from-file`），里面的内容不用转义（转个毛啊转）。

现在来创建Pod吧，快乐的小二逼：

```shell script
cat <<EOF > pod.yaml
apiVersion: v1
kind: List
items:
- kind: Pod
  apiVersion: v1
  metadata:
    name: prod-db-client-pod
    labels:
      name: prod-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: prod-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
- kind: Pod
  apiVersion: v1
  metadata:
    name: test-db-client-pod
    labels:
      name: test-db-client
  spec:
    volumes:
    - name: secret-volume
      secret:
        secretName: test-db-secret
    containers:
    - name: db-client-container
      image: myClientImage
      volumeMounts:
      - name: secret-volume
        readOnly: true
        mountPath: "/etc/secret-volume"
EOF
```

将Pod加到kustomization.yaml文件中：

```shell script
cat <<EOF >> kustomization.yaml
resources:
- pod.yaml
EOF
```

应用这些对象：

```shell script
kubectl apply -k .
```

两个容器在下面这些位置上都会出现对应的文件，其中的值对应着它们各自的生产或测试环境：

```text
/etc/secret-volume/username
/etc/secret-volume/password
```

注意这两个Pod的spec只有一处不同；这就有助于用一个通用的模板来创建不同能力的Pod。

可以通过ServiceAccount进一步简化Pod定义：

- 1.持有`prod-db-secret`的`prod-user`
- 2.持有`test-db-secret`的`test-user`

简化后的Pod定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-db-client-pod
  labels:
    name: prod-db-client
spec:
  serviceAccount: prod-db-client
  containers:
  - name: db-client-container
    image: myClientImage
```

### 用例：Secret数据卷中的隐藏文件

可以在key的前面加一个点，让数据“隐藏”起来。这个key代表了一个点文件（dotfile），或者叫“隐藏”文件。比如挂载下面这个Secret数据卷，`secret-volume`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
  containers:
  - name: dotfile-test-container
    image: k8s.gcr.io/busybox
    command:
    - ls
    - "-l"
    - "/etc/secret-volume"
    volumeMounts:
    - name: secret-volume
      readOnly: true
      mountPath: "/etc/secret-volume"
```

这个数据卷中只包含了一个文件，就是`.secret-file`，在`dotfile-test-container`容器中的路径是`/etc/secret-volume/.secret-file`。

>**注意**：`ls -l`命令看不到以点开头的文件；必须要用`ls -la`才能看清这个世界。

### 用例：只对Pod中一个容器可见的Secret

我们来想象一个处理HTTP请求的程序，包含一些复杂的业务逻辑，然后还要用HMAC对某些消息进行签名。由于其中的逻辑复杂，可能不经意间需要进行远程文件读取，期间可能将私钥泄露给某些恶意的攻击者。

我们可以把它们分成两个容器中的两个进程：一个前端容器，用来和用户交互，做业务逻辑，但是看不到私钥信息；另一个签名容器可以看到私钥，处理由前端发过来的简单的签名请求（比如通过localhost请求）。

通过这种拆分方式，攻击者不能躺着攻击了，而是要开始炫技才有可能攻破我们的大门，比直接读取一个文件要困难很多。

## 最佳实践

### 客户端调用Secret API

当部署了需要用Secret API交互的应用后，应当用[授权策略](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)加以限制，比如[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)。

Secret持有的敏感信息的重要性可能会有多种情况，很多时候可能会导致k8s的权限越界（比如服务账号token），甚至逃逸到集群外部。（这段大部分都是引用了官方中文文档的翻译）。即便个别应用使用Secret的时候能够推断出它的能力有多大，同一个命名空间下的其他应用有可能会将这种努力付之东流。

出于这些原因，对于一个命名空间下的Secret的`watch`和`list`请求就具有非常强大的能力，应当尽量去避免使用，因为如果能将Secret给list出来，那客户端就能看到这个命名空间下的所有Secret的数据。对于集群中所有Secret的`watch`和`list`权限应当保留给具有最高特权、系统级的组件，或者保留给灭霸。

应用要访问Secret API的时候应该使用`get`请求。这样管理员就可以限制对所有Secret的访问，还能给应用设置它们需要的[实例访问白名单](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)。（这段也是引用了官方中文文档的翻译）

如果要提高循环`get`的性能，客户端可以先创建一个引用Secret的资源，然后`watch`这个资源，而不是直接`watch`Secret，当引用发生变化的时候重新请求这个Secret。此外，[“批量watch” API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/bulk_watch.md)，让客户端对多个资源进行独立的`watch`，这种方式我们也提上了日程，很可能发布到未来的版本中。

## 安全

### 保护

因为Secret可以独立于Pod创建，所以在创建、查看、编辑Pod的时候就不那么容易出现相关的泄露风险。而且系统还可以施加一些其他的防御措施，比如避免它们被写入到磁盘上。

只有Pod需要的时候，Secret才会被发送到Pod所在的节点。kubelet会把Secret保存到`tmpfs`，这样就避免被写入到磁盘上。当相关Pod被删除后，kubelet也会同时删除Secret在当地旅行社的副本。

一个节点上可能会有多个Pod的多个Secret。但是只有Pod请求的Secret才会可能被它的容器看到。所以一个Pod根本没可能访问到其他Pod的Secret。

一个Pod中可能会有多个容器。当时每个容器必须通过`volumeMounts`来请求Secret的数据卷，才能看到里面的内容。这就有利于构造[Pod层面的安全分区](#用例：只对Pod中一个容器可见的Secret)。

在大部分的k8s发行版中，用户和apiserver，以及apiserver和kubelet之间的通信都是基于SSL/TLS的。所以Secret在传输的时候也得到了保护。

**功能状态**：`Kubernetes vv1.13 [beta]`

可以为Secret数据开启[REST加密](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)，这样Secret就不会明文保存到[etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)中了。

### 风险

- 在apiserver中，Secret数据是保存在[etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)中的；因此：
  - 管理员应当为集群开启REST数据加密（需要v1.13版本及以上）。
  - 管理员应当限制只能让管理员用户访问etcd。
  - 对于不再被etcd使用的磁盘空间，管理员要及时清理。
  - 如果是etcd集群，管理员最好为节点间通信做上SSL/TLS。
- 如果你是用清单（JSON或YAML）文件创建的Secret，它的数据是经过base64b编码的，如果是分享或者提交到代码仓库了，那Secret就会被压缩。Base64*并不是*加密算法，几乎可以认为是跟明文一样的东西，只不过让外行人看着感觉很牛逼。
- 应用从数据卷中读出Secret的数据后也要做到数据的安全，比如不要冒冒失失地把它们打印到日志文件中，或者传到了某些阴险的第三方程序中。
- 如果一个用户能创建一个引用Secret的Pod，那他就能看到这个Secret中的数据。即便apiserver的策略不允许这个用户读取这个Secret，他也能够通过Pod将Secret的信息暴露出来。
- 目前，只要在任意节点有了root权限，那就能通过apiserver读取到*任意*的Secret，这是因为模拟了kubelet。目前我们计划将Secret只发送到那些真正需要它们的节点上，这样就可以限制某个节点的root用户带来的安全问题。