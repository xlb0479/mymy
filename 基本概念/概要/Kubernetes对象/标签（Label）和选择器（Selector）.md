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

## 标签选择器

跟[名字和UID](对象的名字和ID.md)不一样，标签可没有什么唯一性。而且在实际应用场景中，我们都会给好多对象配置同样的标签。

使用*标签选择器*可以对对象进行筛选。标签选择器是k8s主要的分组方式。

现在的API支持两种选择器：*相等性*和*包含性*。标签选择器可以同时配置多个*条件（requirements）*，用逗号间隔，此时，所有的条件必须同时满足，所以逗号间隔符就有点类似逻辑*与*（`&&`）了。

空值选择器或者不设置选择器，这种情况要看具体的上下文，具体的API文档中应当给出相关的说明。

>**注意**：某些API，比如副本集（ReplicaSet），在同一个命名空间下，其选择器的范围不能重叠，不然你想去吧，A副本集要求X要有3个服务，B副本集要求X要有10000个服务，那到底给多少，这不就乱了么。

>**小心**：不论是哪种类型的选择器，都不支持逻辑*或*（`||`），检查好你的表达式吧。

### *相等性*选择

可以对标签的key和value做*相等*或*不相等*的筛选条件。对于条件中指定的标签，对象必须满足所有的筛选条件，至于条件中没有指定的标签，则没有什么影响。可以使用的操作符有`=`、`==`和`!=`。前两个都是*相等*的意思，最后那个是*不相等*。比如：

```text
environment = production
tier != frontend
```

第一个筛出了所有key为`environment`且对应的值为`production`的资源。第二个筛的是所有key为`tier`且对应的值不等于`frontend`的资源，以及那些没有`tier`这个key的资源。我们还可以查所有`production`下的非`frontend`的资源：
```text
environment=production,tier!=frontend
```

对于相等性条件的使用场景，我再给个例子，就是为Pod指定节点。比如下面的Pod选择了带有“`accelerator=nvidia-tesla-p100`”标签的节点。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### *包含性*选择

*包含*条件可以根据一个集合来筛选。支持三种操作符：`in`、`notin`和`exists`（最后这个操作只能过滤key，不能过滤value）。比如：

```text
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

第一个选择了所有key等于`environment`且值等于`production`或`qa`的资源。第二个选择了所有key等于`tier`且值不等于`frontend`和`backend`，以及那些没有`tier`标签的资源。第三个选择了所有包含标签key为`partition`的资源，不管值是啥。最后一个选择了所有不包含标签key为`partition`的资源，不管值是啥。逗号间隔符同样扮演者逻辑*与*的角色。所以，选择带有标签key为`partition`（不管值是啥），且`environment`不等于`qa`的资源就可以写成`partition,environment notin (qa)`。*包含性*条件相当于*相等性*条件的一般形式，比如你看`environment=production`其实就是`environment in (production)`，`!=`和`notin`也类似。

*包含性*条件可以跟*相等性*条件一起用。比如：`partition in (customerA, customerB),environment!=qa`。

## API

### LIST和WATCH

LIST和WATCH操作可以通过查询参数，使用标签选择器，来过滤返回的对象集合。两种筛选方式都能用（这里给的是URL中的形态）：

- *相等性*条件：

```text
?labelSelector=environment%3Dproduction,tier%3Dfrontend
```

- *包含性*条件：

```text
?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29
```

通过REST客户端，两种选择器都可以用于列表（list）或查看（watch）资源。比如在`kubectl`上使用*相等性*条件：

```text
kubectl get pods -l environment=production,tier=frontend
```

*包含性*条件：

```text
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

*包含性*条件写出来的表达式好像是功能更强大一些，比如可以实现逻辑*或*：

```text
kubectl get pods -l 'environment in (production, qa)'
```

或者使用*存在*操作，限制不匹配的资源：

```text
kubectl get pods -l 'environment,environment notin (frontend)'
```

### API对象的集合引用

一些像是[服务（Service）]()和[副本控制器（ReplicationController）]()之类的对象，用标签选择器来限定它们所管理的资源集合，比如[pod]()。

#### 服务和副本控制器

`服务（Service）`所管理的pod是通过标签选择器来定义的。同样，`副本控制器（ReplicationController）`也是用标签选择器来定义它所管理的pod。

这两种对象的标签选择器可以用`json`或`yaml`文件来定义，只支持*相等性*筛选条件：

```text
"selector": {
    "component" : "redis",
}
```

或

```text
selector:
    component: redis
```

这种选择器（分别是`json`和`yaml`格式）相当于`component=redis`或`component in (redis)`。

#### 支持包含性条件的资源

后来新出的资源，比如[`任务（Job）`]()、[`部署（Deployment）`]()、[`副本集（ReplicaSet）`]()、[`守护进程集（DaemonSet）`]()，都是支持*包含性*筛选的。

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLabels`是由`{key,value}`对组成的映射表。每一个kv对就相当于下面`matchExpressions`中的一个元素，`key`字段就是“key”，`operator`是“In”，`values`数组包含“value”。`matchExpressions`就是一个pod筛选条件列表。可用的操作符有In、NotIn、Exists和DoesNotExist。使用In和NotIn的时候，values数组不能为空。所有的筛选条件，`matchLabels`加上`matchExpressions`，它们要实现逻辑与——必须满足它们列出来的所有条件。

#### 选择节点集合

可以使用标签选择器将Pod限制在某些节点上。详见[节点选择]()。