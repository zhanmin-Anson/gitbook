# YAML文件语法
这篇文章我们来学习使用 Kubernetes 的必备技能：编写配置文件

首先Kubernetes 跟 Docker 等项目的不同就在于它不推荐使用命令行的方式直接运行容器（虽然 Kubernetes 项目也支持这种方式，比如：kubectl run），而是希望你用 YAML 文件的方式，即：把容器的定义、参数、配置，统统记录在一个 YAML 文件中，然后用一句指令把它运行起来，这样做的好处是可以通过文件的方式记录 Kubernetes 做了什么操作和相关的配置。


####一. YAML基础
YAML是专门用来写配置文件的语言，非常简洁和强大，远比JSON格式方便。
YAML语言(发音/jeme/)的设计目标，就是方便人类读写。
#####1. 它的基本语法规则如下：
* 大小写敏感使用
* 缩进表示层级关系
* `缩进时不允许使用Tab键,只允许使用空格`
* 缩进的空格数目不重要,只要相同层级的元素左侧对齐即可
* `#`表示注释,从这个字符一直到行尾，都会被解析器忽略


在kubernetes中，需要了解两种结构类型
 * Maps：字典即key:value 键值对
* Lists： 列表即数组
 

#####2. Maps
Maps是字典，也就是一个 key: value的键值对，Maps可以让我们更加方便的去书写配置信息，例如
```
---
apiVersion: v1
kind: Pod
```
`---`是分隔符，是可选的，在单一文件中可用连续三个`---`区分来多个文件或者资源对象。 上面分别定义两个key(键): kind和 apiVersion，对应的value(值)分别是:V1和Pod。

Maps嵌套
```
---
apiVersion: v1
kind: Pod
metadata:
   name: nginx
   lables:
    app: web
```
metadata这个Key对应的value值就是一个Maps，而且嵌套的Labels这个KEY的值又是一个Maps，我们可以根据自己的情况进行多层嵌套。

#####3. Lists
Lists列表，其实就是数组，在YAML文件中我们可以这样定义
```
args
  - Cat
  - Dog
  - Fish
```
可以有任何数量的项在列表中，每个项的定义以破折号`(-)`开头的与父元素直接可以缩进一个空格。Iist的子项也可以是Maps，Maps的子项也可以是List如下所示
```
---
apiVersion: v1
kind: Pod
metadata:
 name: nginx
 labels:
  app: web
spec:
 containers:
  - name: front-end
    image: nginx
    ports:
     - containerPort: 80
  - name: flaskapp-demo
    image: jcdemo/flaskapp
    ports:
      - containerPort: 5000
```
我们定义了一个叫`containers`的List对象，每个子项都由name，image，ports的组成，每个ports都有一个key为containerPort的map组成。
####二. 使用Yaml创建Kubernetes资源对象
##### 1.  使用Yaml创建创建Pod
```
---
apiVersion: v1
kind: Pod
metadata:
 name: kube100-site
 labels:
  app: web
spec:
 containers:
  - name: front-end
    image: nginx
    ports:
     - containerPort: 80
  - name: flaskapp-demo
    image: jcdemo/flaskapp
    ports:
        - containerPort: 5000
```
 这是我们上面定义的一个普通的POD文件，我们先来简单分析下文件内容
* apiversion：这里它的值是v1，这个版本号需要根据我们安装的 kubernetesk版本资源类型进行变化的，记住不是写死的。
* kind：这里我们创建的是一个Pod，当然根据你的实际情况,这里资源类型可以是 Deployment、Job、 ngress、 Service等。
* metadata：包含了我们定义的Pod的一些meta信息，比如名称namespace、标签等信息。
* spec：包括一些 containers、 storage、volumes，或者其他 Kubernetes需要知道的参数，以及诸如是否在容器失败时重新启动容可以在特定 [KubernetesAPI](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/)找到完整的 Kubernetes Podl的属性。

让我们来看一个典型的容器的定义
```
····
spec：
  containers:
    - name: front-end
      image: nginx
      ports:
        - containerPort: 80
```
这是一个简单的最小定义，名字为( front-end)，基于 nginx的镜像，以及容器将会监听的一个端口(80)。在这些当中，只有名字是非常需要的，你也可以指定一个更加复杂的属性，例如在容器启动时运行的命令，应使用的参数工作目录，或每次实例化时是否拉取映像的新副本。以下为容器可选的设置属性：

* name    
* image
* command
* args
* workingDir
* ports
* env
* resources
* volumeMounts
* livenessProbe
* readinessProbe
* livecycle
* terminationMessage
* imagePullPolicy
* securityContext
* stdin
* stdinOnce
* tty

我们将上面的内容保存为Pod.yaml 文件，来创建Pod资源对象
```
$ kubectl  create -f  pod.yaml
pod "kube 100-site"created
```
查看POD的状态
```
$  kubectl get pods
NAME          READY  STATUS  RESTARTS  AGE
kube-100-site  2/2        Running   0                1m
```
如果状态不是Running，可以使用前面的 kubectl describe进行排查。
我们先删除上面创建的POD:
```
$ kubectl delete  -f pod.yaml
pod "kube 100-site"deleted
```
##### 2.  使用Yaml创建创建Deployment 

在上面的例子中，我们只是单纯的POD实例，但是如果这个POD出现了故障的话，我们的服务也就挂了，所以 kubernetest提供了一个 Deployment的概念，可去管理POD的副本，也就是副本集，这样就可以保证一定数量的副本是可用的，不会以为一个pod挂掉导致整个服务挂掉。我们可以这样定义一个Deployment
```
---
apiVersion : extensions/v1beta1
kind : Deployment
metadata
  name : kube100-site
spec:
  replicas : 2
```
注意:
这里的 apiVersion对应的值是 extensions/1beta1，当然kind要指定为Deployment，然后我们可以指定一些meta信息，比如名字，或者标签之类的。最后最重要的是spec配置选项，这里我们定义需要两个副本，当然还有很多可以设置的属性，比如一个Pod在没有任何错误变成准备的情况下必须达到的最小秒数。

现在我们来定义一个完整的 Deployment的YAML文件
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: kube100-site
spec:
 replicas: 2
 template:
  metadata:
   labels:
    app: web
  spec:
   containers:
    - name: front-end
      image: nginx
      ports:
       - containerPort: 80
    - name: flaskapp-demo
      image: jcdemo/flaskapp
      ports:
       - containerPort: 5000
```
注意其中的 Template,其实就是POD对象的定义。
将上面的YAML文件保存为 deployment.yaml,然后创建Deployment
```
 $ kubectl  create   -f  deployment.yaml
```
同样的,想要查看它的状态,我们可以检查 Deployment的列表
```
$ kubectl get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube100-site   2         2         2            2           3d
```
我们可以看到所有的Pods都已经正常运行。


总结：到这里我们就完成了使用YAML文件创建 Kubernetes Deployment的过程，在创建其他的资源对象时也是如此，在了解了YAML文件的基础后，定义YAML文件其实已经很简单了，最主要的是要根据实际情况去定义YAML文件，所以查阅 Kubernetes文档很重要可以使用[http://www.yamllint.com/](http://www.yamllint.com/)去检验YAML文件的合法性。

---------------------------------------------------------------------------------------------------------------
上篇文章：[k8s二 | 使用kubeadm部署K8S](https://www.jianshu.com/p/dea93dea5a4e)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)
参考资料：[从Docker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)
---------------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-bf8ad73af7f806b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



