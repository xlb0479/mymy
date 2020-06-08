# CronJob

**功能状态**：`Kubernetes v1.8 [beta]`

*CronJob*可以创建重复调度的[Job](Job—这个屌丝总是有始有终.md)。

一个CronJob对象类似于*crontab*（cron table）文件中的一行。他会按照一个指定的频率周期性的运行一个任务，并且采用了[Cron](https://en.wikipedia.org/wiki/Cron)格式来定义。

>**警告**：
>所有的**CronJob**`schedule：`时间度量都是基于[kube-controller-manager]()的时区来定的。
>如果control plane将kube-controller-manager运行在Pod或普通容器中，kube-controller-manager所在容器中的时区就决定了CronJob调度所使用的时区。

创建CronJob的时候，要确保你使用的名字是有效的[DNS子域名](../../概要/Kubernetes对象/对象的名字和ID.md#DNS子域名)。名字长度不能超过52个字符。这个长度限制是因为CronJob控制器会自动为Job名称添加一个11个字符的后缀，而Job名字的长度限制则是63个字符。

- [CronJob](#CronJob)
- [限制](#限制)
- [下一步……](#下一步)

## CronJob

CronJob常用于周期性循环执行的任务，比如数据备份或邮件发送。也可以在指定时间上执行一个单独的任务，比如在集群可能空闲的时候调度一个Job。

### 栗子

下面的CronJob栗子会在每分钟打印当前时间以及一条问候消息：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

（[用CronJob实现自动化任务执行]()）中介绍了这个例子的更多细节。

## 限制

一个CronJob会在每次调度的时候创建一个*左右*的Job对象。我们说“左右”是因为有些情况下可能会创建两个Job，或者根本不创建Job。我们会尽量减少这种情况出现，但是无法完全避免。因此Job应当是*幂等的*。

如果把`startingDeadlineSeconds`设置成一个很大的值，或者不设置（默认），并且`concurrencyPolicy`等于`Allow`，Job的执行永远都是保证在至少一个。

对于每个CronJob，CronJob[控制器](../../集群架构/控制器.md)会检查从上一次调度至今一共丢失了多少次调度。如果丢失次数超过了100次，它就不会再启动Job了，并且记录一个错误日志

```text
Cannot determine if job needs to be started. Too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.
```

需要注意的是，如果你给`startingDeadlineSeconds`字段设置了值（非`nil`），控制器计算丢失Job是根据`startingDeadlineSeconds`至当前时间进行计算的。也就是说，比如`startingDeadlineSeconds`是`200`，每次就是计算过去200秒内丢了多少个Job。

当CronJob在本该调度的时间却创建失败，那就认为这个Job丢失了。比如把`concurrencyPolicy`设置为`Forbid`，当CronJob调度触发，但是上一次调度依然在执行中的话，这次的Job就丢掉了。

举个栗子，比如从`08:30:00`开始CronJob需要每分钟调度一个新的Job，没有设置`startingDeadlineSeconds`。假设CronJob控制器在`08:29:00`到`10:21:00`这段时间挂掉了，那么Job就不会再启动了，因为丢失的Job数量已经超过了100。

为了进一步解释这个东西，我们再次假设一个CronJob从`08:30:00`开始每分钟要调度一个新的Job，并且`startingDeadlineSeconds`设置为200秒。而CronJob控制器恰巧又在同样的时间段挂掉了（`08:29:00`到`10:21:00`），那么新的Job会在10:22:00开始运行。这是因为此时的控制器只会计算过去200秒丢失的Job数量（这里是3个），而不是从上次调度开始计算的。

CronJob只是负责在需要调度的时间创建Job，而Job负责管理它的Pod。

## 下一步……

[Cron表达式](https://pkg.go.dev/github.com/robfig/cron?tab=doc#hdr-CRON_Expression_Format)，用于CronJob的`schedule`字段。
关于如何创建和使用CronJob，具体的栗子，去看[使用CronJob运行自动化任务]()。