# StatefulSet
这篇文章主要介绍Pod如何通过Deployment的控制器ReplicatSet实现水平扩展与滚动更新。
#### 一. 控制器模式
在kubernetes项目中的设计思想是“控制器”模式，在前面文章[k8s(一) 基本概念与组件原理](https://www.jianshu.com/p/c231c7132fae)中介绍的`controller manager`组件就是一系列控制器的集合，我们可以通过 Kubernetes 项目的 pkg/controller 目录查看这些控制器：
```
$ cd kubernetes/pkg/controller/
$ ls -d */              
deployment/             job/                    podautoscaler/          
cloud/                  disruption/             namespace/              
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
```
这个目录下面的每一个控制器，都以独有的方式负责某种编排功能。
以部署Nginx-Deployment为例：
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
上面Yaml文件定义的`spec.replicas`字段为2，即确保携带了` app=nginx `标签的 Pod 的个数永远等于2个，这就是我们定义的期望状态，意味着如果集群中超过两个副本就会删除旧的Pod，反之就会有新的Pod创建。实现方式大概如下：

1. Deployment 控制器从 Etcd 中获取到所有携带了`app: nginx`标签的 Pod，然后统计它们的数量，这就是实际状态；
2. Deployment 对象的 Replicas 字段的值就是期望状态，即我们事先定义的副本数；
3. Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod。

Deployment 这种控制器的设计原理，就是用一种对象管理另一种对象的方式。

类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象`PodTemplate`模板组成。
![unmin.club](https://upload-images.jianshu.io/upload_images/9644303-eb1015312cc92953.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 二. 作业副本的水平扩展/收缩
在上面例子中简单介绍了Deployment控制器的实现方式，实现了 Kubernetes 项目中一个非常重要的功能：Pod 的“水平扩展 / 收缩”（horizontal scaling out/in）。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。

举个例子，如果你更新了 Deployment 的 Pod 模板（比如，修改了容器的镜像），那么 Deployment 就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。而这个能力的实现，依赖的是 Kubernetes 项目中的一个非常重要的概念（API 对象）：ReplicaSet。

Deployment 实际控制的是ReplicaSet对象，由ReplicaSet来管理Pod，
![unmin.club](https://upload-images.jianshu.io/upload_images/9644303-a1791440e54433a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过上图，我们可以看到定义了 replicas=3 的 Deployment，与它的 ReplicaSet，以及 Pod 的关系，实际上是一种“层层控制”的关系。Deployment 通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。

**实现水平扩展 / 收缩**
Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以实现水平扩展 / 收缩。我们可以通过修改YAML文件或者kubectl scale来具体实现，比如将上面的nginx-deployment的副本数扩展为4：
```
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```
#### 三. 滚动更新
#####1.  滚动更新的实现原理
我们根据上面例子中的nginx-deployment来进一步的理解滚动更新的实现过程。
```
$ kubectl create -f nginx-deployment.yaml --record
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```
`--record`  记录下你每次操作所执行的命令，以方便后面查看。
返回结果中四个字段分别对应的含义如下：
* DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；
* CURRENT：当前处于 Running 状态的 Pod 的个数；
* UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致；
* AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

下面我们通过`kubectl rollout status` 来实时查看资源对象的状态变化
```
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```
从返回结果中可以看到已经有2个Pod进入了UP-TO-DATE状态，稍微等待一下后查看Deployment的状态
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           20s
```
可以看到3个Pod都已经变成了AVAILABLE 状态，然后再查看这个Deployment控制的ReplicaSet
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-3167673210   3         3         3       20s
```
根据上面流程我们可以看到，当提交了创建Deployment请求后，Deployment Controller 就会立即创建一个 Pod 副本个数为 3 的 ReplicaSet，其中ReplicaSet 的 DESIRED、CURRENT 和 READY 字段的含义，和 Deployment 中是一致的。所以，相比之下，Deployment 只是在 ReplicaSet 的基础上，添加了 UP-TO-DATE 这个跟版本有关的状态字段。

现在来修改Deployment的Pod模板来触发滚动更新
可以直接使用`kubectl edit`命令或者通过修改YAML文件更新资源
```

$ kubectl edit deployment/nginx-deployment
... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -> 1.9.1
        ports:
        - containerPort: 80
...
deployment.extensions/nginx-deployment edited
```
保存退出，Kubernetes 就会立刻触发“滚动更新”的过程。通过  `kubectl rollout status` 指令查看 nginx-deployment 的状态变化：
```
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.extensions/nginx-deployment successfully rolled out
```
可以通过查看Deployment 的 Events事件信息，看到这个“滚动更新”的流程
```
$ kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
...
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 0
```
通过事件日志我们可以清楚的看到滚动更新的具体流程即：
1. Deployment Controller 使用这个修改后的 Pod 模板，创建一个新的ReplicaSet（hash=1764197365），这个新的 ReplicaSet 的初始 Pod 副本数是：0。
2. Deployment Controller 开始将这个新的 ReplicaSet 所控制的 Pod 副本数从 0 个变成 1 个，即：“水平扩展”出一个副本。
3. Deployment Controller 又将旧的 ReplicaSet（hash=3167673210）所控制的旧 Pod 副本数减少一个，即：“水平收缩”成两个副本。

直到旧ReplicaSet的Pod副本数为0，新ReplicaSet的副本数为3，则意味着滚动更新完成。完成后我们可以查看RS的状态：
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   3         3         3       6s
nginx-deployment-3167673210   0         0         0       30s
```

可以看到Hash为3167673210的旧ReplicaSet已经被水平伸缩为0个副本。

将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。

#####2.  滚动更新的配置

为了进一步保证服务的连续性，Deployment Controller 还会确保，在任何时间窗口内，只有指定比例的 Pod 处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新 Pod 被创建出来。这两个比例的值都是可以配置的，默认都是 DESIRED 值的 25%。


所以，在上面这个 Deployment 例子中，它有 3 个 Pod 副本，那么控制器在“滚动更新”的过程中永远都会确保至少有 2 个 Pod 处于可用状态，至多只有 4 个 Pod 同时存在于集群中。这个策略，是 Deployment 对象的一个字段，名叫 `RollingUpdateStrategy`，如下所示：
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
在上面这个` RollingUpdateStrategy `的配置中，`maxSurge `指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod；而 `maxUnavailable` 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod。这两个配置还可以百分比形式来表示，比如：maxUnavailable=50%，指的是我们最多可以一次删除“50%*DESIRED 数量”个 Pod。

结合以上讲述，现在我们可以扩展一下 Deployment、ReplicaSet 和 Pod 的关系图
![unmin.club](https://upload-images.jianshu.io/upload_images/9644303-871f57c2d34a1bf8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上所示，Deployment 的控制器，实际上控制的是 ReplicaSet 的数目，以及每个 ReplicaSet 的属性。而一个应用的版本，对应的正是一个 ReplicaSet；这个版本应用的 Pod 数量，则由 ReplicaSet 通过它自己的控制器（ReplicaSet Controller）来保证。通过这样的多个 ReplicaSet 对象，Kubernetes 项目就实现了对多个“应用版本”的描述。

##### 3. 对应用进行版本控制
在日常的代码更新中，我们都是通过版本来控制代码的上线，但上线过程中难免会有升级失败的情况，下面我们来实践操作下Deployment对应用进行版本控制的具体流程，方便我们了接具体的实现原理。

首先我们通过`kubectl  set image`的指令来修改nginx-depolyment使用的镜像，把原先的镜像替换为错误的镜像，来模拟升级失败。
```
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated
```
由于这个 nginx:1.91 镜像在 Docker Hub 中并不存在，所以这个 Deployment 的“滚动更新”被触发后，会立刻报错并停止。这时，我们来检查一下 ReplicaSet 的状态，如下所示：
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   2         2         2       24s
nginx-deployment-3167673210   0         0         0       35s
nginx-deployment-2156724341   2         2         0       7s
```
现在新版本的 ReplicaSet（hash=2156724341）已经创建了两个 Pod，但是它们都没有进入 READY 状态。这当然是因为这两个 Pod 都拉取不到有效的镜像。与此同时，旧版本的 ReplicaSet（hash=1764197365）的“水平收缩”，也自动停止了。此时，已经有一个旧 Pod 被删除，还剩下两个旧 Pod。

现在我们可以通过`kubectl rollout undo`命令将Deployment回滚到上一个版本。
```
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
```
如果我们需要回滚到指定版本的话，可以通过` kubectl rollout history`来查看历史变更的版本，这是因为前面我们创建的时候添加`--record`参数，所以都会被记录下来。
```
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```
可以看到变更次数为三次，失败的则为版本3，现在我们通过`--to-revision=2` 指定回滚到版本2。
```
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
```
当发布的版本次数过多时，Deployment同样会创建很多的ReplicaSet，我们可以通过修改 `spec.revisionHistoryLimit`参数，来控制 Kubernetes 为 Deployment 保留的“历史版本”个数。

总结：现在我们通过实践知道了kubernetes设计思想实际是通过一个个控制器来具体实现的，也了解了Deployment 这个 Kubernetes 项目中最基本的编排控制器的实现原理和使用方法。Deployment 实际上是一个两层控制器。首先，它通过 ReplicaSet 的个数来描述应用的版本；然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。

---------------------------------------------------------------------------------------------------------------
上篇文章：[k8s四 | 深入理解Pod资源对象](https://www.jianshu.com/p/d1e1de86d6d6)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)
参考资料：[深入剖析Kubernetes-张磊](https://time.geekbang.org/column/article/40366)
---------------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-bf8ad73af7f806b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


