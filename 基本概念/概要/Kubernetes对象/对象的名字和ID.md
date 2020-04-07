# 对象的名字和ID

每个对象都有个[*名字*](#名字们)，对于它所属的资源类型来说，是唯一的。对于整个集群来说，每个对象还有个[*UID*](#UID)，这个是在集群内唯一的。

这么说吧，在同一个[命名空间]()下，只能有一个Pod叫`myapp-1234`，但是可以有同名的Deployment。

要是想自定义一些不唯一的属性，可以用[标签（label）]()和[注解（annotation）]()。

- [名字们](#名字们)
- [UID](#UID)
- [接下来……](#接下来)

## 名字们

名字就是个URL字符串，比如`/api/v1/pods/some-name`。

同一种类型下，对象名字是唯一的。你要是把某个对象删了，可以重建相同名字的对象。

下面列出了几类常见的命名规范。

### DNS子域名

好多资源的名字都可以用作DNS子域名（[RFC1123](https://tools.ietf.org/html/rfc1123)）。这也就是说：

- 长度不能大于253
- 只能包含小写字母、数字、“-”或“.”
- 以字母或数字开头
- 以字母或数字结尾

### DNS标签名

一些资源名称需要遵循DNS标签标准（[RFC1123](https://tools.ietf.org/html/rfc1123)）。这也就是说：

- 最多63个字符
- 只能包含小写字母和数字，以及“-”
- 以小写字母或数字开头
- 以小写字母或数字结尾

### 路径段名称（Path Segment Names）

一些资源要求它们的对象名都够当作某个路径中的一段，也就是说，名字不能是“.”、“..”，不能包含“/”、“%”。

下面是一个叫`nginx-demo`的Pod。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

>**注意**：有些资源对名字还有其他约束。

## UID

由k8s系统生成，用于唯一标识对象。

集群中每个对象都会有一个唯一的UID。这样，相似的实体，即便以前出现过，也能区分出来。

k8s的UID是通用唯一标识（也叫UUID）。UUID这玩意已经标准化了。

## 接下来……

- 看看k8s的[标签]()。
- 看看[k8s的标识与名称](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md)的设计文稿。