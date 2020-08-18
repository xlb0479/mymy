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

#### Secret解码

### Secret作为Pod中的文件

### Secret作为环境变量

### 用作imagePullSecrets