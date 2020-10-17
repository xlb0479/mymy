# Operator模式

Operator是k8s的软件扩展，用[自定义资源](扩展k8s的API/自定义资源.md)来管理应用和它们的组件。Operator遵守k8s的原则，尤其是[控制循环](../集群架构/控制器.md)。

## 动机

Operator模式涵盖了那些管理服务的操作人员重点关注的内容。负责维护特定应用和服务的操作员知道他的系统应该要如何工作，如何部署，以及出现问题时会有怎样的表现。

用k8s来跑应用的人们通常会用一些自动化工作来完成重复性的任务。Operator模式可以让你来编写代码实现k8s本身并没有提供的自动化任务。

## k8s中的Operator

k8s本身就是被设计成自动化的。k8s核心提供了大量内置的开箱即用的自动化工作。你可以用k8s自动化完成服务的部署和运行，*而且*，你还可以自动化控制k8s如何进行这些工作。

k8s的[控制器](../集群架构/控制器.md)概念能让你无需修改k8s的代码就能扩展集群的行为。Operator是k8s API的客户端，扮演了[自定义资源](扩展k8s的API/自定义资源.md)的控制器。

## 一个Operator栗子

使用Operator可以实现的自动化包括：

- 按需部署一个应用
- 备份和恢复应用的状态
- 处理应用代码的升级以及相关的变更，比如数据库schema或其他配置的变更
- 为无法支持k8s API发现的应用发布一个Service
- 模拟集群的部分或整体异常来验证它的弹性
- 无需内部的成员选举进程，为分布式应用选取leader

再详细一点来看，一个Operator像是什么呢？下面的栗子更加详细：

- 1.一个名为SampleDB的自定义资源，你可以将它配置到集群中。
- 2.一个Deployment运行了一个Pod，Pod中包含了这个Operator的控制器的部分。
- 3.Operator代码的容器镜像。
- 4.控制器代码查询control plane，查出配置了什么样的SampleDB资源。
- 5.这个Operator的核心部分是告诉apiserver如何让一切满足已配置的资源。
    - 如果你新增了一个SampleDB，Operator要为其安装PVC来提供持久化数据库存储，一个StatefulSet来运行SampleDB，以及一个Job来处理初始化配置。
    - 如果你把它删了，Operator要做快照，确保StatefulSet和数据卷都删了。
- 6.Operator还要负责数据库的常规备份。对于每一个SampleDB资源，Operator决定着何时创建一个Pod连到数据库上然后开始做备份。这些Pod依赖一个ConfigMap和/或一个Secret，其中包含了数据库连接的信息和凭证。
- 7.因为Operator要为它管理的资源提供健壮的自动化工作，所以还会有其他的支撑代码。对于本例，就类似于要有代码检查数据库是否还运行着一个旧的版本，如果是的话，那就要创建Job对象，帮你做升级。

## 部署Operator

最常用的Operator部署方法就是添加CRD及其相关的控制器。控制器一般都是运行在[control plane](https://v1-18.docs.kubernetes.io/docs/reference/glossary/?all=true#term-control-plane)外部，跟任意的容器化应用差不多。比如你可以用Deployment来运行控制器。

## 使用Operator

Operator部署后，你就可以用它来添加、修改或删除对应的资源。接着上面的栗子，你可以为Operator本身创建一个Deployment，然后：

```shell script
kubectl get SampleDB                   # 查询配置好的数据库

kubectl edit SampleDB/example-database # 手动更新一些配置
```

……然后大功告成了！Operator会负责应用这些变更，并且保证服务的良好运行。

## 编写你自己的Operator

如果已有的的Operator无法满足你的需要，你可以自己写一个。在[下面](#下一步)的内容中你会找到一些库和工具，可以用它们来编写你自己的云原生Operator。

你可以用任意语言/运行时来实现Operator（控制器），只要它能作为一个[k8s API客户端](https://v1-18.docs.kubernetes.io/docs/reference/using-api/client-libraries/)就行。

## 下一步……

- 学习[自定义资源](扩展k8s的API/自定义资源.md)
- 在[Operatorhub.io](https://operatorhub.io/)中找到适合你的现成的Operator
- 用现有工具来编写自己的Operator，比如：
    - 使用[KUDO](https://kudo.dev/)（Kubernetes Universal Declarative Operator）
    - 使用[kubebuilder](https://book.kubebuilder.io/)
    - 使用[Metacontroller](https://metacontroller.app/)搭配你自己实现的WebHook
    - 使用[Operator框架](https://github.com/operator-framework/getting-started)
- 把你的Operator[发布](https://operatorhub.io/)给其他人用
- 阅读[CoreOS的原始文献](https://coreos.com/blog/introducing-operators.html)，它引入了Operator模式
- 阅读一篇来自Google Cloud的[文章](https://cloud.google.com/blog/products/containers-kubernetes/best-practices-for-building-kubernetes-operators-and-stateful-apps)，介绍了构建Operator的最佳实践