# Pod预设值

本文介绍PodPresets，它可以用来在Pod创建的时候注入一些信息。这些信息包括secret、数据卷、卷挂载以及环境变量。

- [理解Pod预设值](#理解Pod预设值)
- [它是咋工作的](#它是咋工作的)
- [启用Pod预设值](#启用Pod预设值)
- [接下来……](#接下来)

## 理解Pod预设值

`Pod Preset`也是一种API资源，用来在Pod创建的时候为其注入一些额外的运行时需求。使用[标签选择器](../../概要/Kubernetes对象/标签（Label）和选择器（Selector）.md#标签选择器)为PodPreset指定对应的Pod。

使用PodPreset，可以让用户不必为每个Pod都定义全部的信息。此时，Pod模板的编写者在使用一个服务时，就不需要知道这个服务的全部细节了。

关于这玩意的来由，可以看一下它的[设计思想](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/service-catalog/pod-preset.md)。

## 它是咋工作的

