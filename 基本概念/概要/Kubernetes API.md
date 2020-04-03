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

## API版本
## API分组
## 开启/关闭API分组
## 开启extensions/v1beta1分组中特定的资源