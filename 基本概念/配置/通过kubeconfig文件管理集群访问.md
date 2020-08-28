# 通过kubeconfig文件管理集群访问

kubeconfig文件可以组织集群、用户、命名空间以及认证机制的相关信息。`kubectl`命令根据kubeconfig文件中的信息来选择要连接的集群，以及和apiserver进行通信。

>**注意**：用于配置集群访问的文件叫做*kubeconfig文件*。它就是指的某个配置文件，而不是说文件的名字叫`kubeconfig`。

默认情况下，`kubectl`会在`$HOME/.kube`目录下查找一个名为`config`的文件。可以通过`KUBECONFIG`环境变量或者是[`--kubeconfig`](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)参数来制定其他的kubeconfig配置文件。

关于如何创建和设置kubeconfig文件，详见[配置多集群访问](https://v1-18.docs.kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)。

## 多集群、多用户以及多种认证机制

假设你有多个集群吧，用户和认证机制也有很多种。比如：

- 某个kubelet使用证书认证。
- 某个用户使用token授权。
- 管理员可能给每个用户发放了不同的证书。

有了kubeconfig文件，你可以将集群、用户和命名空间组织起来。可以定义上下文，在多个集群和命名空间之间快速的切换。

## 上下文

kubeconfig文件中的`context`元素可以将访问参数组织在一个好认的名字下。每个上下文有三个参数：cluster、namespace和user。默认情况下，`kubectl`命令使用`current context`中的参数来连接集群。

要设置当前的上下文：

```shell script
kubectl config use-context
```

## KUBECONFIG环境变量

`KUBECONFIG`环境变量包含了一个kubeconfig文件列表。对Linux和Mac系统来说，是冒号间隔的。对Windows，是分号间隔。`KUBECONFIG`环境变量不是必须的。如果没有，`kubectl`就是用默认的kubeconfig文件，`$HOME/.kube/config`。

如果有，`kubectl`使用的配置是将`KUBECONFIG`中设置的文件列表合并后的结果。

## 合并kubeconfig文件

要想看一下你的配置，执行：

```shell script
kubectl config view
```

正如上面所述，命令输出的内容可能来自一个文件，也可能来自多个kubeconfig文件整合后的结果。

下面是`kubectl`命令在合并kubeconfig文件时的合并规则：

- 1.如果设置了`--kubeconfig`参数，就只用参数指定的文件。不会发生合并。该参数只能有一个。

    否则，如果设置了`KUBECONFIG`环境变量，将它作为一个要准备合并的文件列表。合并规则如下：

    - 忽略空文件名。
    - 如果文件内容无法反序列化，报错。
    - 对于某个值，哪个文件先设置了，哪个文件就说了算。
    - 绝不对值或者映射中的key进行修改。比如：谁先设置了`current-context`，上下文就确定了。比如：如果两个文件都有`red-user`，只取第一个文件中的`red-user`。即便第二个文件中在`red-user`下还有完全不冲突的内容，也一并忽略。

    关于如何设置`KUBECONFIG`环境变量，详见[设置KUBECONFIG环境变量](https://v1-18.docs.kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable)。
    
    再否则，就是用默认的`$HOME/.kube/config`，不会产生任何合并。
    
- 2.看看谁先设置了上下文，就用谁的：
    - 1.如果有`--context`命令行参数，那就用这个。
    - 2.使用合并后的kubeconfig文件中的`current-context`。
    
  这里可以允许空的上下文。
  
- 3.选择集群和用户。此时，可能有，也可能没有上下文。下面的流程要跑两遍：一次选用户，一次选集群：
    - 1.如果设置了`--user`或者`--cluster`命令行参数，那就用这个。
    - 2.如果上下文非空，用上下文中的用户或集群。
    
  这里可以允许空的用户和集群。

- 4.选择最终要使用的集群信息。此时，可能有，也可能没有集群信息。根据下面的流程拼凑出集群的信息；依然是谁先出现就用谁：
    - 1.优先使用命令行参数：`--server`、`--certificate-authority`、`--insecure-skip-tls-verify`。
    - 2.如果某些集群信息的属性出现在合并后的kubeconfig文件中，那就把它们加上。
    - 3.如果没有服务器地址，报错。
- 5.选择最终要使用的用户信息。跟上一步差不多，只不过每个用户只能设置一种认证机制：
    - 1.优先使用命令行参数：`--client-certificate`、`--client-key`、`--username`、`--password`、`--token`。
    - 2.根据合并后的kubeconfig文件来设置`user`字段。
    - 3.如果出现两种冲突的认证机制，报错。
- 6.如果还缺少什么信息，那就使用默认值，需要授权信息的话就弹出相关提示。

## 文件引用

kubeconfig文件中的文件和路径引用是相对于kubeconfig文件位置的。命令行中的文件引用是相对于当前工作目录的。在`$HOME/.kube/config`中，相对路径就是相对路径，绝对路径就是绝对路径。吃屎就是吃屎。

## 下一步……

- [配置多集群访问](https://v1-18.docs.kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
- [`kubectl config`](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)