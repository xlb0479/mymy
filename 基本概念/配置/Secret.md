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

### Secret作为环境变量

### 用作imagePullSecrets