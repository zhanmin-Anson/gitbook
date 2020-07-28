# DaemonSet
这篇文章主要介绍Kubernetes中第三个重要编排对象DaemonSet守护进程的实现原理及使用方法。
## 一. DaemonSet 简介
DaemonSet：服务守护进程，它的主要作用是在Kubernetes集群的所有节点中运行我们部署的守护进程，相当于在集群节点上分别部署Pod副本，如果有新节点加入集群，Daemonset会自动的在该节点上运行我们需要部署的Pod副本，相反如果有节点退出集群，Daemonset也会移除掉部署在旧节点的Pod副本。

###1. DaemonSet的主要特征：
* 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
* 每个节点上只会运行一个这样的 Pod 实例；
* 如果新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；
* 而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

###2. DaemonSet常用场景：
* 网络插件的 Agent 组件，如（Flannel，Calico）需要运行在每一个节点上，用来处理这个节点上的容器网络；
* 存储插件的 Agent 组件，如（Ceph，Glusterfs）需要运行在每一个节点上，用来在这个节点上挂载F远程存储目录；
* 监控系统的数据收集组件，如（Prometheus Node Exporter，Cadvisor）需要运行在每一个节点上，负责这个节点上的监控信息搜集。
* 日志系统的数据收集组件，如（Fluent，Logstash）需要运行在每一个节点上，负责这个节点上的日志信息搜集。

## 二. DaemonSet的实现原理
  DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。比如在创建Kubernetes集群后，Node节点上由于没有可用的容器网络，集群节点的状态会是NotReady，普通的Pod将无法运行，我们就需要通过DaemonSet 部署一个网络插件的 Agent 组件。下面我们来了解下DaemonSet如何设计实现的。

###1. DaemonSet是如何确保每个节点只运行一个Pod？

 1. DaemonSet的控制器模型`DaemonSet Controller`先从从 Etcd 里获取所有的 Node 列表；
2. 然后遍历所有的 Node检查，当前这个 Node节点上是不是有一个携带了我们定义标签的 Pod 在运行；
2. 如果没有定义的 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
3. 如果有定义的 Pod，但是数量大于 1，那就说明要调用 Kubernetes API 把多余的 Pod 从这个 Node 上删除掉；
4. 如果正好只有一个定义的 Pod，那说明这个节点是正常的。
###2 . 如何只在指定的节点上运行Pod？
首先我们会想到使用`nodeSelector`字段来指定Node的名字，但是在 Kubernetes 项目里，`nodeSelector` 其实已经是一个将要被废弃的字段了。因为，现在有了一个新的、功能更完善的字段可以代替它，即：`nodeAffinity`。·举个例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```
上面文件定义了` spec.affinity `字段，它是 Pod 里和调度相关的一个字段，然后又定义了一个nodeAffinity（节点关系）。这里它的定义含义是：
-   requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；
- 这个 Pod，将来只允许运行在“metadata.name”是“node-geektime”的节点上;
- operator: In（即：部分匹配；如果你定义 operator: Equal，就是完全匹配）

>DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象资源里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。当然，DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向 Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象。

###3. 污染与容忍
在经过上面的流程后，DaemonSet 还会给这个 Pod 自动加上另外一个与调度相关的字段，叫作` tolerations`（容忍）。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint），即可以在有污点的节点调度运行，继而保证每个节点上都会被调度一个 Pod。

DaemonSet 自动添加的` tolerations 字段`，格式如下所示：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```
举个例子：在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为`node.kubernetes.io/network-unavailable`的“污点”。而通过这样一个 `Toleration`，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。
```yaml
...
template:
    metadata:
      labels:
        name: network-plugin-agent
    spec:
      tolerations:
      - key: node.kubernetes.io/network-unavailable
        operator: Exists
        effect: NoSchedule
```

尽管DaemonSet Pod遵守[污染(taint)和容忍(toleration)](http://www.coderdocument.com/docs/kubernetes/v1.14/concepts/configuration/taints_and_tolerations.html)，但以下容忍会根据相关特性自动添加到DaemonSet 管理的Pod中。

| Toleration Key | 影响 | 版本 | 描述 |
| :-- | :-- | :-- | :-- |
| `node.kubernetes.io/not-ready` | NoExecute | 1.13+ | 当存在节点问题(如网络分区)时，DaemonSet pod不会被驱逐。 |
| `node.kubernetes.io/unreachable` | NoExecute | 1.13+ | 当存在节点问题(如网络分区)时，DaemonSet pod不会被驱逐。 |
| `node.kubernetes.io/disk-pressure` | NoSchedule | 1.8+ |  |
| `node.kubernetes.io/memory-pressure` | NoSchedule | 1.8+ |  |
| `node.kubernetes.io/unschedulable` | NoSchedule | 1.12+ | 在默认的调度程序中，DaemonSet pod允许不可调度的属性。 |
| `node.kubernetes.io/network-unavailable` | NoSchedule | 1.12+ | 使用主机网络的DaemonSet pod，默认调度器允许网络不可用属性。 |
>K8S的调度策略还有很多，会在后面文章中专门介绍

##三. DaemonSet的使用方法

现在我们通过DaemonSet部署日志收集组件Fluentd，具体的来了解DaemonSet的使用方法。
DaemonSet管理的是一个 fluentd-elasticsearch 镜像的 Pod，
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```


可以看到，DaemonSet 跟 Deployment 其实非常相似，只不过是没有 replicas 字段，
它也使用 `selector `选择管理所有携带了` name=fluentd-elasticsearch `标签的 Pod。而这些 Pod 的模板，也是用 `template` 字段定义的。在这个字段中，我们定义了一个使用 fluentd-elasticsearch:1.20 镜像的容器，这个镜像通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中。而且这个容器挂载了两个 hostPath 类型的 Volume，分别对应宿主机的 /var/log 目录和 /var/lib/docker/containers 目录，然后也定义了容忍master上的污点。

>需要注意的是，Docker 容器里应用的日志，默认会保存在宿主机的 /var/lib/docker/containers/容器ID-json.log 文件里，所以这个目录正是 fluentd 的搜集目标。还有我们一般在创建DaemonSet对象时都应该加上` resources`字段进行资源的限制，防止它占用过多的宿主机资源。

创建资源对象
```shell
$ kubectl create -f fluentd-elasticsearch.yaml
$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   2         2         2         2            2           <none>          1h
$ kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY     STATUS    RESTARTS   AGE
fluentd-elasticsearch-dqfv9   1/1       Running   0          53m
fluentd-elasticsearch-pf9z5   1/1       Running   0          53m
```
创建资源对象后，可以看到启动了两个Pod对象，这是因为DaemonSet会根据集群的节点来控制Pod的数量，有几个节点就会起相应的几个Pod对象，确保每个节点都有一个pod对象运行。

## 四. DaemonSet的版本管理

查看DaemonSet历史版本
```shell
$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
```
更新DaemonSet中的镜像版本到 v2.2.0
```shell
$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
```
>`--record` ：更新使用到的命令会出现在 DaemonSet 的 rollout history 历史版本里面。

查看镜像更新状态
```shell
$ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 of 2 updated pods are available...
daemon set "fluentd-elasticsearch" successfully rolled out
```
Deployment 管理这些版本，靠的是“一个版本对应一个 ReplicaSet 对象”。而DaemonSet 控制器操作的直接就是 Pod，所以不会有 ReplicaSet 这样的对象参与其中。

那么，它的这些版本又是如何维护的呢？所谓，一切皆对象！在 Kubernetes 项目中，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现。当然，“版本”也不例外。Kubernetes v1.7 之后添加了一个 API 对象，名叫 `ControllerRevision`，专门用来记录某种 Controller 对象的版本，ControllerRevision 其实是一个通用的版本管理对象。这样，Kubernetes 项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。

查看 `fluentd-elasticsearch `对应的 ControllerRevision：
```shell
$ kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-64dc6799c9   daemonset.apps/fluentd-elasticsearch   2          1h
```
查看当前controllerrevision的事件信息
```shell
$ kubectl describe controllerrevision fluentd-elasticsearch-64dc6799c9 -n kube-system
Name:         fluentd-elasticsearch-64dc6799c9
Namespace:    kube-system
Labels:       controller-revision-hash=2087235575
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation=2
              kubernetes.io/change-cause=kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
...
Revision:                  2
Events:                    <none>
```

可以看到，这个 ControllerRevision 对象，实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令。

回滚DaemonSet历史版本
```shell
$ kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
```
这个 kubectl rollout undo 操作，实际上相当于读取到了` Revision=1 `的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象。

总结：在这篇文章中主要介绍了Kubernetes中第三个重要的编排对象DaemonSet，相比于 Deployment，DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration 这两个调度器的小功能，保证了每个节点上有且只有一个 Pod。与此同时，DaemonSet 通过使用 ControllerRevision，来保存和管理自己对应的“版本”。其中StatefulSet编排对象也是使用 ControllerRevision 进行版本管理的，这是因为在 Kubernetes 项目里，ControllerRevision 其实是一个通用的版本管理对象。

---------------------------------------------------------------------------------------------------------------
上篇文章：[k8s六 | 理解有状态应用StatefulSet](https://www.jianshu.com/p/55fafd29e329)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)
参考资料：[深入剖析Kubernetes-张磊](https://time.geekbang.org/column/article/40366)
---------------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-bf8ad73af7f806b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


