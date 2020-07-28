# 基本概念与组件原理
>参考资料：[从Docker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)


# 一. 什么是kubernetes？
[kubernetes](https://kubernetes.io/zh/docs/)是一个可移植的，可扩展的开源平台，是Google开源的容器集群管理系统（谷歌内部:Borg)，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。

# 二. 为什么使用kubernetes？
k8s在Docker技术的基础上，为容器化的应用提供部署运行、资源调度、服务发现和动态伸缩等一系列完整功能，提高了大规模容器集群管理的便捷性。同时Kubernetes是一个完备的分布式系统支撑平台，具有完备的集群管理能力，多扩多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和发现机制、內建智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制以及多粒度的资源配额管理能力。同时Kubernetes提供完善的管理工具，涵盖了包括开发、部署测试、运维监控在内的各个环节。
#  三. 集群架构及组件
## 1. 集群架构
![公众号: 前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-68fd2a486214d371?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **Master**
负责管理集群，部署集群所需组件`etcd，apiserver，controller manager，scheduler`。master 协调集群中的所有活动，例如调度应用程序、维护应用程序的所需状态、扩展应用程序和滚动更新。

 **Node**
节点是 Kubernetes 集群中的工作节点，用于托管正在运行的应用程序，可以是物理机或虚拟机。 每个工作节点都有一个 `kubelet`和`kube-proxy`，它是管理节点并与 Kubernetes Master 节点进行通信的代理。节点上还应具有处理容器操作的容器运行时，例如 Docker 或 rkt。一个 Kubernetes 工作集群至少有三个节点。

## 2. 集群组件
* etcd： 键值存储数据库，维护集群内各个节点状态的一致性，保存集群的状态及配置；
* apiserver：处理资源操作的请求，并提供认证、授权、访问控制、API 注册和发现等机制；
* controller manager： 控制器管理，负责维护集群的状态，如故障检测、自动扩展、滚动更新等；
* scheduler：调度器，负责资源的调度，按照预定的调度策略将 Pod 调度到相应的节点；
* kubelet：负责维护容器的生命周期， Pod 的创建、启动、监控、重启、销毁等工作，处理Master节点下发到本节点的任务；
* Container runtime： 负责镜像管理以及 Pod 和容器的真正运行（CRI）；
* kube-proxy： 负责为 Service 提供 cluster 内部的服务发现和负载均衡；
* Flannel/calico：网络插件， 负责为整个集群提供 IP 服务；
* kube-dns/coredns： 负责为整个集群提供 DNS 服务；
* Ingress Controller： 为服务提供外网入口；
# 四. 集群工作流程
集群各组件的通信原理，以创建Pod为例：
![公众号：前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-87ad770fac8f5f5b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 使用REST API 创建Pod，即(kubectl create pod)请求apiserver。
2. apiserver接收到pod创建请求后，写入到Etcd，会存在记录但不会创建。
3. scheduluer 检测到有未绑定 Node 的 Pod，查找集群中资源充足的Node绑定，并将调度信息写入到Etcd。
4. kubelet 通过监测etcd数据库，检测到有绑定该节点的Pod调度过来需要创建，调用container runtime 运行该 Pod。
5. kubelet 通过 container runtime 取到 Pod 状态，并更新到 apiserver 中。
# 五. 基本概念
Kubernetes中的绝大部分概念都会被抽象成Kubernetes管理的一种资源对象，下图为k8s资源对象全景图

![公众号：前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-c86138f893b91e15?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1. 相关名词
### NameSpace
Namespace 命名空间是对一组资源和对象的抽象集合， 是 Linux 内核用来隔离内核资源的方式。NameSpace做隔离，Cgroups 做限制，rootfs 做文件系统。

### Label
Label 标签以 key/value 的方式附加到资源对象上如Pod， 其他对象可以使用 Label Selector 来选择一组相同 label 的对象。
## 2.编排对象
### Pod
Pod是 Kubernetes 项目中最小的 API 资源对象，Pod可以由一个或多个业务容器和一个根容器(Pause容器)组成。一个Pod表示某个应用的一个实例。Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的，凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。

可以这样理解，云计算系统的操作系统是 k8s ，容器就相当于是其进程，而 Pod 则是进程组，容器镜像就是这个系统里的“.exe”安装包。Pod 里的所有容器，它们共享PID、IPC、Network和UTS namespace，可以声明共享同一个 Volume。
###  ReplicaSet
 ReplicaSet是Pod副本的抽象，用于解决Pod的扩容和伸缩。
### Deployment
Deployment通常用来部署无状态应用，如Web服务， 该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的。在内部使用ReplicaSet来实现Pod副本的创建。Deployment确保指定数量的Pod“副本”在运行，并且支持回滚和滚动升级。
创建Deployment时，需要指定 Pod模板和Label标签。
### StatefulSet
StatefulSet通常用来部署有状态应用，如Mysql服务，服务运行的实例需要在本地存储持久化数据，多个实例之间有依赖拓扑关系，比如：主从关系、主备关系。如果停止掉依赖中的一个Pod，就会导致数据丢失或者集群崩溃。他的核心功能就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。它包含Deployment控制器ReplicaSet的所有功能，增加可以处理Pod的启动顺序，为保留每个Pod的状态设置唯一标识，同时具有以下功能：
* 稳定的、唯一的网络标识符
* 稳定的、持久化的存储
* 有序的、优雅的部署和缩放
### DaemonSet
DaemonSet：服务守护进程，它的主要作用是在Kubernetes集群的所有节点中运行我们部署的守护进程，相当于在集群节点上分别部署Pod副本，如果有新节点加入集群，Daemonset会自动的在该节点上运行我们需要部署的Pod副本，相反如果有节点退出集群，Daemonset也会移除掉部署在旧节点的Pod副本。

DaemonSet的主要特征：
* 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
* 每个节点上只会运行一个这样的 Pod 实例；
* 如果新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；
* 而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

DaemonSet常用场景：
* 网络插件的 Agent 组件，如（Flannel，Calico）需要运行在每一个节点上，用来处理这个节点上的容器网络；
* 存储插件的 Agent 组件，如（Ceph，Glusterfs）需要运行在每一个节点上，用来在这个节点上挂载F远程存储目录；
* 监控系统的数据收集组件，如（Prometheus Node Exporter，Cadvisor）需要运行在每一个节点上，负责这个节点上的监控信息搜集。
* 日志系统的数据收集组件，如（Fluent，Logstash）需要运行在每一个节点上，负责这个节点上的日志信息搜集。

### Job/CronJob
* Job负责处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束，解决一些需要进行批量数据处理和分析的需求，，比如Jenkins Slave，发布完代码后任务结束，Pod自动销毁；
* CronJob则就是在Job上加上了时间调度，用来执行一些周期性的任务。
###  HPA
Horizontal Pod Autoscaling（Pod水平自动伸缩），简称HPA。通过监控分析RC或者Deployment控制的所有Pod的负载变化情况来确定是否需要调整Pod的副本数量，这是HPA最基本的原理。
## 3. 其他对象
###  ConfigMap
ConfigMap：就是为了让镜像和配置文件解耦，以便实现镜像的可移植性和可复用性，因为一个configMap其实就是一系列配置信息的集合，将来可直接注入到Pod中的容器使用，而注入方式有两种，一种将configMap做为存储卷，一种是将configMap通过env中configMapKeyRef注入到容器中；
### RBAC
RBAC：基于角色的访问控制，可以用来给用户授予对集群操作不同的权限。
### Secret
Secret：用来保存敏感信息，例如密码、OAuth 令牌和 ssh key等等，将这些信息放在Secret中比放在Pod的定义中或者docker镜像中来说更加安全和灵活。
## 4. 服务发现
###  Service
Service：是一种抽象的对象，它定义了一组Pod的逻辑集合和一个用于访问它们的策略，我们可以通过访问Service来访问到后端的Pod服务，其实这个概念和微服务非常类似。一个Serivce下面包含的Pod集合一般是由Label Selector来决定的。
###  Ingress
Ingress：就是从 kuberenets 集群外部访问集群的一个入口，将外部的请求转发到集群内不同的 Service 上，其实就相当于 nginx、haproxy 等负载均衡代理服务器，目前选择有很多： traefik、nginx-controller、Kubernetes Ingress Controller for Kong、HAProxy Ingress controller。
## 5. 存储对象
### PV/PVC
* PV 的全称是：PersistentVolume（持久化卷），是对底层的共享存储的一种抽象，PV 由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接。
* PVC 的全称是：PersistentVolumeClaim（持久化卷声明），PVC 是用户存储的一种声明，PVC 和 Pod 比较类似，Pod 消耗的是节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正使用存储的用户不需要关心底层的存储实现细节，只需要直接使用 PVC 即可。
### StorageClass
StorageClass：动态 PV，可以自动帮我们创建 PV，不再需要手动创建PV。

## 6. 其他概念
### Helm
Helm：包管理工具，相当于kubernetes环境下的yum包管理工具。
### CRD
CRD是对 Kubernetes API 的扩展，Kubernetes 中的每个资源都是一个 API 对象的集合，例如我们在YAML文件里定义的那些spec都是对 Kubernetes 中的资源对象的定义，所有的自定义资源可以跟 Kubernetes 中内建的资源一样使用 kubectl 操作。

### Operator
Operator是由CoreOS公司开发的，用来扩展 Kubernetes API，特定的应用程序控制器，它用来创建、配置和管理复杂的有状态应用，如数据库、缓存和监控系统。Operator基于 Kubernetes 的资源和控制器概念之上构建，但同时又包含了应用程序特定的一些专业知识，比如创建一个数据库的Operator，则必须对创建的数据库的各种运维方式非常了解，创建Operator的关键是CRD（自定义资源）的设计。Operator是将运维人员对软件操作的知识给代码化，同时利用 Kubernetes 强大的抽象来管理大规模的软件应用。

---------------------------------------------------------------------------------------------------------------
下篇文章：[使用kubeadm快速部署K8S集群](https://blog.csdn.net/qq_39254455/article/details/105729732)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)


-----------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-0b4ae39913673411?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



