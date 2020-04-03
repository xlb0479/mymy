# Kubernetes API
详细的API约定见[API约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)。

API端点（endpoint）、资源类型、例子可以在[API参考手册]()中找到。

关于API远程访问，见[API远程访问控制]()。

k8s的API为声明式配置提供了基础。可以用[kubectl]()命令来创建、修改、删除和获取API对象。

k8s依据各项API资源，对其自身的状态进行序列化存储（目前是存到[etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/)里了）。

k8s把自己拆成了各种组件，组件之间通过API交互。

- [API变更](#API变更)
- [OpenAPI和Swagger定义](#OpenAPI和Swagger定义)
- [API版本](#API版本)
- [API分组](#API分组)
- [开启/关闭API分组](#开启/关闭API分组)
- [开启extensions/v1beta1分组中特定的资源](#开启extensions/v1beta1分组中特定的资源)

## API变更
以吾之见，任何成功的系统，都要随着客观事物的发展而不断变化，潇洒走一回，不枉此生。所以我们希望Kubernetes API也能达到这种目标，但是我们在发展的过程中还要考虑到向后兼容的问题。一般来说，如果我们要新增一些API资源或者属性字段，那问题不大，但要是想删除或者停用某个资源/属性，那就得遵守[API废弃策略]()。

如何修改API，如何保证兼容性，这些细节见[API更新文档](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md)。

## OpenAPI和Swagger定义
所有API都使用了[OpenAPI](https://www.openapis.org/)规范。

从1.10开始，Kubernetes API通过`/openapi/v2`端点提供OpenAPI接口。请求格式由HTTP头来确定：

Header|值
-|-
Accept|`application/json`, `application/com.github.proto-openapi.spec.v2@v1.0+protobuf`（如果你传`*/*`或者不传，那默认为`application/json`）
Accept-Encoding|`gzip`（不传也中）

就在1.14之前吧，端点名称中就区分了格式（`/swagger.json`，`/swagger-2.0.0.json`，`/swagger-2.0.0.pb-v1`，`/swagger-2.0.0.pb-v1.gz`）。这种端点已经废弃了，在1.14中已经被删掉了。

**OpenAPI规范示例**

1.10之前|从1.10开始
-|-
GET /swagger.json|GET /openapi/v2 **Accept**: application/json
GET /swagger-2.0.0.pb-v1|GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf
GET /swagger-2.0.0.pb-v1.gz|GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf Accept-Encoding: gzip

k8s实现了基于ProtoBuf序列化的API，主要是为了集群内部通信使用，具体内容可以看[protobuf设计方案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md)，以及API对象的Go源码中的IDL文件。

1.14之前吧，apiserver提供了一个`/swaggerapi`接口，可以获取Kubernetes API的[Swagger v1.2]()定义。这个接口也废了，1.14里已经删了。

## API版本

为了便于废弃属性或者重构资源定义，k8s支持多版本API，每个版本有不同的路径，比如`/api/v1`、`/apis/extensions/v1beta1`。

之所以没有在资源或者属性层面上区分版本，是为了让API能够表达一个清晰的、一致的系统资源和行为的视图，同时也可以对那些已经完犊子的或者正处于试验期间的API开启访问控制。JSON和ProtoBuf序列化也遵循着同样的方针政策，以下内容都是同时涵盖这两种格式的。

API版本和软件本身的版本之间并没有非常直接的关联关系。[API与软件版本化方案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md)中做了详细描述。

不同的API版本意味着不同的稳定性和支持程度。具体等级详见[API更新文档](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions)，这里吾辈总结如下：

- Alpha版本：
   - 版本号中包含`alpha`字样（比如`v1alpha1`）。
   - 开启这些功能的时候可能有bug。默认不开启。
   - 可能会随时删掉，而且根本不会告诉你。
   - 后续版本中，这种API可能出现不兼容的情况，而且根本不会告诉你。
   - 由于以上原因，建议只用于小型的试验环境。
- Beta版本：
   - 版本号中包含`beta`字样（比如`v2beta3`）。
   - 都测过了。可以安全使用。默认开启。
   - 不会删掉，但细节可能发生变化。
   - 对象的schema和语义，在后续的beta或stable版本中，可能会出现不兼容的问题。出现这种问题的话，我们会告诉你该如何升级。这时候可能要涉及到API对象的删除、编辑和重建。其中，编辑的时候，你可得长点儿心。整个过程中，应用可能需要停服。
   - 由于以上原因，建议只在非关键业务场景中使用。如果你有好几个集群，每个集群可以单独升级，那你可以不用这么谨慎，但也别浪。
   - **希望大家踊跃尝试beta版本的功能并发表你们的意见！一旦这些功能退出了beta版本，那我们就不会再改了。**
- Stable版本（稳定版）：
   - 版本号是`vX`，`X`是个整数。
   - 稳定版的功能会发布到正式版软件中，并且，会活得很久。

## API分组
为了更方便的扩展Kubernetes API，我们实现了[***API分组***](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/api-group.md)。API分组用REST路径的形式声明在对象的`apiVersion`中。

当前已经有一些API分组了：

1. ***core***分组，也被称为***legacy***分组，REST路径为`/api/v1`，使用`apiVersion: v1`。

2. 其他的命名分组的REST路径格式为`/apis/$GROUP_NAME/$VERSION`，并使用`apiVersion: $GROUP_NAME/$VERSION`（比如`apiVersion: batch/v1`）。完整的API分组见[Kubernetes API参考手册]()。

通过[自定义资源]()，有两种方式可以对API进行扩展：

1. 使用[CustomResourceDefinition]()。

2. 可以完全自己使用apiserver，通过[aggregator]()实现无缝衔接，釜底抽薪。

## 开启/关闭API分组

默认情况下已经给你开启了一部分资源和API分组。可以通过apiserver的`--runtime-config`参数来设置它们的开启或关闭。这个参数的值是一个逗号间隔值。比如，要关闭batch/v1，就设置为`--runtime-config=batch/v1=false`，要开启batch/v2alpha1，就设置为`--runtime-config=batch/v2alpha1`。可以用逗号间隔的多个k=v进行设置。

>**注意**：修改`--runtime-config`参数之后要重启apiserver和controller-manager。

## 开启extensions/v1beta1分组中特定的资源

`extensions/v1beta1`分组中的DaemonSet、Deployment、StatefulSet、NetworkPolicy、PodSecurityPolicy以及ReplicaSet默认都是关闭的。比如要开启Deployment和DaemonSet，就要这么设置一下：`--runtime-config=extensions/v1beta1/deployments=true,extensions/v1beta1/daemonsets=true`。

>**注意**：只有`extensions/v1beta1`分组下的资源支持单独开关，这也是由于一些遗留问题导致的。