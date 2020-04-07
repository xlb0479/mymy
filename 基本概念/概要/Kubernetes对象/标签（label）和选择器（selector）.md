# 标签（label）和选择器（selector）

*标签*就是对象上的一些kv格式的属性。标签么，就是打标签，给对象增加一些辨识度，增加一些有意义的标记，跟底层系统没什么关系。标签可以用来组织或筛选对象。随时都可以打标签，一开始就加上，或者后续再加，都行，随时修改。每个对象都可以定义标签。标签的每个key，在其所属的对象上必须是唯一的。

```text
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

可以在UI界面或命令行工具上，通过标签，来实现快速的查询。对于那些不是用来区分对象的信息，应当使用[注解（annotation）]()来记录。

- [为啥要弄这么个东西](#为啥要弄这么个东西)
- [规范](#规范)
- [标签选择器](#标签选择器)
- [API](#API)

## 为啥要弄这么个东西

标签，可以让用户以一种松耦合的方式来自由的组织对象，而且对于客户端来说，并不需要存储这些映射关系。

服务部署和批处理流水线，它们通常都涉及到多维度的实体（多分区，多部署，多版本，多层次，每个层次中又有多个微服务）。管理时经常需要交叉操作，打破了严格的层次化的封装，特别是那些由基础设施而不是由用户决定的极其严格的层次划分。

一些例子：

- `"release" : "stable"`、`"release" : "canary"`
- `"environment" : "dev"`、`"environment" : "qa"`、`"environment" : "production"`
- `"tier" : "frontend"`、` "tier" : "backend"`、` "tier" : "cache"`
- `"partition" : "customerA"`、` "partition" : "customerB"`
- `"track" : "daily"`、` "track" : "weekly"`

上面列了一些常用的使用场景；这些个玩意你都可以自由发挥。记住，一个对象中每个标签的key必须是唯一的。

## 规范

*标签*是键值对。标签的key分两段：可选的前缀和具体的名字，用斜线间隔（`/`）。名字长度不能超过63个字符，开头结尾必须是字符或数字（`[a-z0-9A-Z]`），中间可以用减号（`-`）、下划线（`_`）、点（`.`）间隔。前缀是可选的，前缀必须满足DNS子域名的规范：点（`.`）间隔的DNS标签，总长度不超过253个字符，结尾加斜线（`/`）。

如果没加前缀，那这个key就被认为是用户私有的。为终端用户提供标签的自动化系统组件（比如`kube-scheduler`、`kube-controller-manager`、`kube-apiserver`、`kubectl`以及其他第三方自动化程序），必须要设置前缀。

`kubernetes.io/`和`k8s.io/`这两个前缀是为k8s核心组件保留的。

标签的值，规则跟名字一样，我就不重复了，不同的是，值可以为空。

下面的Pod闹了两个标签，`environment: production`和`app: nginx`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```