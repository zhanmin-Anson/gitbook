# 深入理解Pod
这篇文章我们来深入了解Pod的基本概念及相关使用

####一. Pod的设计思路
首先Pod是 Kubernetes 项目中最小的 API 对象，而Pod也是由容器组组成的。Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的，而容器就成为Pod属性中一个普通的字段定义。

我们可以这样理解：容器是相当于未来云计算的进程，镜像是安装包，Pod则是传统环境的机器，k8s是操作系统。pod是一个小家庭，它把密不可分的家庭成员(container)聚在一起，Infra container则是家长，掌管家中共通资源，家庭成员通过sidecar方式互帮互助，其乐融融～。
#### 二. Pod的网络通信Infra容器
在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。这样的组织关系，可以用下面这样一个示意图来表达：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-8efe7dfc0c6358d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图所示，这个 Pod 里有两个用户容器 A 和 B，还有一个 Infra 容器。很容易理解，在 Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。而在 Infra 容器“Hold 住”Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了

* 这也就意味着，对于 Pod 里的容器 A 和容器 B 来说：
* 它们可以直接使用 localhost 进行通信；
* 它们看到的网络设备跟 Infra 容器看到的完全一样；
* 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
* 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
* Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。


而对于同一个 Pod 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 Infra 容器完成的。这一点很重要，因为将来如果你要为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。


这就意味着，如果你的网络插件需要在容器里安装某些包或者配置才能完成的话，是不可取的：Infra 容器镜像的 rootfs 里几乎什么都没有，没有你随意发挥的空间。当然，这同时也意味着你的网络插件完全不必关心用户容器的启动与否，而只需要关注如何配置 Pod，也就是 Infra 容器的 Network Namespace 即可。


####三. 伴生容器与容器初始化
* sidecar： 伴生容器，指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。
* Init Container：初始化容器 就是用来做初始化工作的容器，可以是一个或者多个，如果有多个的话，这些容器会按定义的顺序依次执行，只有所有的Init Container执行完后，主容器才会被启动。我们知道一个Pod里面的所有容器是共享数据卷和网络命名空间的，所以Init Container里面产生的数据可以被主容器使用到的。

例如：我们现在有一个 Java Web 应用的 WAR 包，它需要被放在 Tomcat 的 webapps 目录下运行起来。假如，你现在只能用 Docker 来做这件事情，那该如何处理这个组合关系呢？
* 一种方法是，把 WAR 包直接放在 Tomcat 镜像的 webapps 目录下，做成一个新的镜像运行起来。可是，这时候，如果你要更新 WAR 包的内容，或者要升级 Tomcat 镜像，就要重新制作一个新的发布镜像，非常麻烦。
* 另一种方法是，你压根儿不管 WAR 包，永远只发布一个 Tot 容器。不过，这个容器的 webapps 目录，就必须声明一个 hostPath 类型的 Volume，从而把宿主机上的 WAR 包挂载进 Tomcat 容器当中运行起来。不过，这样你就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有 WAR 包的目录呢？这样来看，你只能独立维护一套分布式存储系统了。

实际上，有了 Pod 之后，这样的问题就很容易解决了。我们可以把 WAR 包和 Tomcat 分别做成镜像，然后把它们作为一个 Pod 里的两个容器“组合”在一起。这个 Pod 的配置文件如下所示：
```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```
在这个 Pod 中，我们定义了两个容器，第一个容器使用的镜像是geektime/sample:v2，这个镜像里只有一个 WAR 包（sample.war）放在根目录下。而第二个容器则使用的是一个标准的 Tomcat 镜像。
 

不过，你可能已经注意到，WAR 包容器的类型不再是一个普通容器，而是一个 Init Container 类型的容器。在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

所以，这个 Init Container 类型的 WAR 包容器启动后，我执行了一句"cp /sample.war /app"，把应用的 WAR 包拷贝到 /app 目录下，然后退出。而后这个 /app 目录，就挂载了一个名叫 app-volume 的 Volume。接下来就很关键了。Tomcat 容器，同样声明了挂载 app-volume 到自己的 webapps 目录下。所以，等 Tomcat 容器启动时，它的 webapps 目录下就一定会存在 sample.war 文件：这个文件正是 WAR 包容器启动时拷贝到这个 Volume 里面的，而这个 Volume 是被这两个容器共享的。


像这样，我们就用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。实际上，这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。

在我们的这个应用 Pod 中，Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用 Init Container初始化容器的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色。
	
####四. Pod的生命周期状态
* Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
* Running。这个状态下，Pod已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
* Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
* Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
* Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。


####五. Pod的钩子Hook
 Kubernetes 为我们的容器提供了生命周期钩子的，就是我们说的Pod Hook，Pod Hook 是由 kubelet 发起的，当容器中的进程启动前或者容器中的进程终止之前运行，这是包含在容器的生命周期之中。我们可以同时为 Pod 中的所有容器都配置 hook。

Kubernetes 为我们提供了两种钩子函数：

* PostStart：这个钩子在容器创建后立即执行。但是，并不能保证钩子将在容器ENTRYPOINT之前运行，因为没有参数传递给处理程序。主要用于资源部署、环境准备等。不过需要注意的是如果钩子花费太长时间以至于不能运行或者挂起， 容器将不能达到running状态。
* PreStop：这个钩子在容器终止之前立即被调用。它是阻塞的，意味着它是同步的， 所以它必须在删除容器的调用发出之前完成。主要用于优雅关闭应用程序、通知其他系统等。如果钩子在执行期间挂起， Pod阶段将停留在running状态并且永不会达到failed状态。
如果PostStart或者PreStop钩子失败， 它会杀死容器。所以我们应该让钩子函数尽可能的轻量。当然有些情况下，长时间运行命令是合理的， 比如在停止容器之前预先保存状态。

另外我们有两种方式来实现上面的钩子函数：

* Exec - 用于执行一段特定的命令，不过要注意的是该命令消耗的资源会被计入容器。
* HTTP - 对容器上的特定的端点执行HTTP请求
示例
```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```
在这个例子中，我们在容器成功启动之后定义了Lifecycle字段，即在 /usr/share/message 里写入了一句“欢迎信息”（即 postStart 定义的操作）。而在这个容器被删除之前，我们则先调用了 nginx 的退出指令（即 preStop 定义的操作），从而实现了容器的“优雅退出”

####六. Pod的健康检查
在 Kubernetes 中，你可以为 Pod 里的容器定义一个健康检查“探针”（Probe）。这样，kubelet 就会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器进行是否运行（来自 Docker 返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。
* liveness probe（存活探针）：确定你的应用程序是否正在运行，通俗点将就是是否还活着。一般来说，如果你的程序一旦崩溃了， Kubernetes 就会立刻知道这个程序已经终止了，然后就会重启这个程序。而我们的 liveness probe 的目的就是来捕获到当前应用程序还没有终止，还没有崩溃，如果出现了这些情况，那么就重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去。
* readiness probe（可读性探针）：确定容器是否已经就绪可以接收流量过来了。这个探针通俗点讲就是说是否准备好了，现在可以开始工作了。只有当 Pod 中的容器都处于就绪状态的时候 kubelet 才会认定该 Pod 处于就绪状态，因为一个 Pod 下面可能会有多个容器。当然 Pod 如果处于非就绪状态，那么我们就会将他从我们的工作队列(实际上就是我们后面需要重点学习的 Service)中移除出来，这样我们的流量就不会被路由到这个 Pod 里面来了，决定的这个 Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期。。

探针支持的配置方式：
* exec：执行一段命令
* http：检测某个 http 请求
* tcpSocket：检查端口， kubelet 将尝试在指定端口上打开容器的套接字。如果可以建立连接，容器被认为是健康的，如果不能就认为是失败的。

示例
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
在这个 Pod 中，我们定义了一个容器。它在启动之后做的第一件事，就是在 /tmp 目录下创建了一个 healthy 文件，以此作为自己已经正常运行的标志。而 30 s 过后，它会把这个文件删除掉。与此同时，我们定义了一个这样的 livenessProbe（健康检查）。它的类型是 exec，这意味着，它会在容器启动后，在容器里面执行一句我们指定的命令，比如：“cat /tmp/healthy”。这时，如果这个文件存在，这条命令的返回值就是 0，Pod 就会认为这个容器不仅已经启动，而且是健康的，如果返回值为非0，那么kubelet将会重启这个容器。`initialDelaySeconds:5`，在容器启动 5 s 后开始执行健康检查，`periodSeconds: 5`每 5 s 执行一次。

创建这个Pod查看过程
```
$ kubectl  get pods 
test-liveness-exec                                                1/1     Running            0          64s
```
30s后我们查看Pod的Event
```
$ kubectl  describe pod test-liveness-exec
Events:
  Type     Reason     Age                 From                 Message
  ----     ------     ----                ----                 -------
  Normal   Scheduled  2m5s                default-scheduler    Successfully assigned default/test-liveness-exec to k8s-node01
  Warning  Unhealthy  66s (x3 over 76s)   kubelet, k8s-node01  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```
通过Event发现健康检查探查到 /tmp/healthy 已经不存在了，所以它报告容器是不健康的，我们在看一下Pod的当前状态是否正常？
```
$ kubectl   get pods 
NAME                                                              READY   STATUS             RESTARTS   AGE
test-liveness-exec                                                1/1     Running            1          1m1s
```
我们可以看到Pod的状态仍然是Running正常的，但是`RESTARTS`字段已经由0已经变成1了，这是因为Pod已经被kubelet重启了。

livenessProbe 也可以定义为发起 HTTP 或者 TCP 请求的方式，定义格式如下：
```
livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
       initialDelaySeconds: 3
       periodSeconds: 3
```
```
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```
 Pod 其实可以暴露一个健康检查 URL（比如 /healthz），或者直接让健康检查去检测应用的监听端口。这两种配置方法，在 Web 服务类的应用中非常常用。
####七. Pod的故障恢复机制
上面例子中健康检查如果没有通过，kubelet则会重启这个容器，这是因为默认Pod的恢复机制为Always，其实它是 Pod 的 Spec 部分的一个标准字段`（pod.spec.restartPolicy）`，我们可以根据自己的需求来定通过设置 restartPolicy改变 Pod 的恢复策略：
* Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
* OnFailure: 只在容器 异常时才自动重启容器；
* Never: 从来不重启容器。

基本的设计原理:
1. 只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod 就会保持 Running 状态，并进行容器重启。否则，Pod 就会进入 Failed 状态 。

2. 对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数

####八. Pod的常用属性定义
除了上文定义的一些属性字段，我们常用的属性还有以下字段定义：
* NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段
* ImagePullPolicy：[Always | Never | IfNotPresent] # 【String】 每次都尝试重新拉取镜像 | 仅使用本地镜像 | 如果本地有镜像则使用，没有则拉取
* Volume：volume是Pod中多个容器访问的共享目录。volume被定义在pod上，被这个pod的多个容器挂载到相同或不同的路径下。一般volume被用于持久化pod产生的数据。Kubernetes提供了众多的volume类型，包括emptyDir、hostPath、nfs、glusterfs、cephfs、ceph rbd等。
* NodeName：一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。
* HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容。

网上资料：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-fa874853571b1fde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9644303-03714ace6fccc00f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9644303-0effac4dcbe75ed9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9644303-629f97164c4c9533.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9644303-05784ecbdfa2c621.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/9644303-928f5f7b942b1330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
总结：在这篇文章中我们深度了解了Pod的设计模式和字段属性定义，其实Pod 这个看似复杂的 API 对象，实际上就是对容器的进一步抽象和封装而已，Pod 对象就是容器的升级版。它对容器进行了组合，添加了更多的属性和字段。
---------------------------------------------------------------------------------------------------------------
上篇文章：[k8s三 | 使用YAML文件定义资源对象](https://www.jianshu.com/p/294892bef857)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)
参考资料：[深入剖析Kubernetes-张磊](https://time.geekbang.org/column/article/40366)、[从Docker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)
---------------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-bf8ad73af7f806b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




