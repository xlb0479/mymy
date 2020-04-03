#理解Kubernetes对象
本文向你介绍如何用KubernetesAPI来表达k8s对象，以及它们在`.yaml`文件中的格式。

- [理解k8s对象](#理解k8s对象)
- [接下来……](#接下来)

## 理解k8s对象
***k8s对象***是k8s系统中的持久化实体。k8s用这些实体来描述集群的状态。特别是以下几点，都注意听：

- 运行了哪些容器（运行在哪些节点上）
- 这些应用都用了哪些资源
- 关于应用的各种策略，重启策略、升级和容错策略

k8s对象是一种“意图记录”（啧啧，高深了啊这就）——当你创建一个对象，k8s通过不断的运作(花点儿钱，搞搞关系)，保证对象一直存在。如何创建对象呢？你只需要将工作负载描述给k8s系统就可以了；也就是传说中的***目标状态（desired state）***。

操作k8s对象的时候——不论是创建、修改还是删除——你都要用到[Kubernetes API](../Kubernetes%20API.md)。当你用`kubectl`的时候，这个命令会帮你调用所需的Kubernetes API。当然你也可以在你自己的程序中使用[客户端SDK]()直接调用Kubernetes API。

### 对象声明（Spec）与状态（Status）
几乎所有的k8s对象都包含两个嵌套对象，掌管着对象的配置：***`spec`***对象和***`status`***对象。对于包含`spec`的对象，在创建对象的时候你就必须设置这个属性，描述你所需的资源特性：即***目标状态***。

`status`则描述了对象的**当前状态（current state）*，由k8s和它的组件来设置和更新。[control plane]()天天兴奋的跟个什么似的，就是为了让每个对象的实际状态保持在你提供的目标状态上。

比如吧，在k8s中，一个Deployment就是一个对象，描述了一个运行在集群上的应用。你创建Deployment的时候就需要设置它的`spec`，声明你这个应用要跑3个副本。k8s读取了这份Deployment声明，为你的应用启动了3个实例——更新状态以匹配你的声明。如果某个实例挂了（状态变了），系统会发现这种声明和状态的差异并做出响应——对于此例，就是启动一个替代的实例。

关于对象声明、状态以及元数据的更多信息，见[Kubernetes API约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)。

### 描述一个k8s对象

创建对象的时候吧，你要提供对象声明，描述目标状态，以及对象的一些基本信息（比如名字，连名字都没有，你能干成个啥）。当你用Kubernetes API创建对象的时候（直接调用或使用`kubectl`），API会将这些信息闹成JSON，放到请求体里面。**更常见的场景是将这些信息写在.yaml文件中然后发给`kubectl`**。`kubectl`调用API的时候会把这些信息闹成JSON。

下面给了一个Deployment的对象声明，包含了必须的属性：

>[application/deployment.yaml](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/deployment.yaml)

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

创建Deployment的一种方式就是用像上面这样的`.yaml`文件，把它当作[`kubectl apply`]()命令的一个参数：

```text
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
```

运行后结果如下：

```text
deployment.apps/nginx-deployment created
```

### 必需的属性
在创建k8s对象的`.yaml`文件中，你需要设置以下几个属性：

- `apiVersion`——你使用哪个版本的Kubernetes API来创建对象
- `kind`——要创建哪种对象
- `metadata`——用来唯一标识对象，包括`name`、`UID`以及可选的`namespace`
- `spec`——描述对象的目标状态

详细的`spec`格式对于每个k8s对象来说都是不一样的，还包含一些对象特定的嵌套属性。[Kubernetes API参考手册]()中包含了所有对象的声明格式。比如Pod的`spec`格式在[PodSpec v1 core]()，Deployment的`spec`格式在[DeploymentSpec v1 apps]()。

## 接下来……

- 学习[Kubernetes API概要]()，了解更多API的概念
- 学习k8s的基本对象，比如[Pod]()
- 学习k8s的[控制器（Controller）]()