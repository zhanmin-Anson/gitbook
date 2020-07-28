# ServiceAccount
## 1. ServiceAccount 介绍
首先Kubernetes中账户区分为：User Accounts（用户账户） 和 Service Accounts（服务账户） 两种，它们的设计区别如下：
- UserAccount是给kubernetes集群外部用户使用的，例如运维或者集群管理人员，使用kubectl命令时用的就是UserAccount账户；UserAccount是全局性。在集群所有namespaces中，名称具有唯一性，默认情况下用户为admin；
- ServiceAccount是给运行在Pod的程序使用的身份认证，Pod容器的进程需要访问API Server时用的就是ServiceAccount账户；ServiceAccount仅局限它所在的namespace，每个namespace都会自动创建一个default service account；创建Pod时，如果没有指定Service Account，Pod则会使用default Service Account。

## 2. 默认Service Account
在上篇文章 [k8s九 | 详解配置对象ConfigMap与Secret](https://www.jianshu.com/p/31f841ad6e61)最后kubernetes.io/service-account-token一节中我们已经提到创建命名空间时会创建一个默认的Service Account，而ServiceAccout 创建时也会创建对应的 Secret，下面我们实际操作下。

创建命名空间
```shell
$ kubectl  create  ns  anmin
namespace/anmin created
```
查看命名空间的ServiceAccount
```shell
$ kubectl  get sa -n anmin
NAME      SECRETS   AGE
default   1         27s
```
查看ServiceAccount的Secret
```shell
$ kubectl  describe  sa  default   -n anmin
Name:                default
Namespace:           anmin
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-bskds
Tokens:              default-token-bskds
Events:              <none>
$ kubectl  get  secret -n anmin
NAME                  TYPE                                  DATA   AGE
default-token-bskds   kubernetes.io/service-account-token   3      75s
```
可以看到在创建名为“anmin”的命名空间后，自动创建了名为“default”的ServiceAccount，并为“default”服务账户创建了对应kubernetes.io/service-account-token类型的secret。

创建一个Pod
```shell
apiVersion: v1
kind: Pod
metadata:
  name: testpod
  namespace: anmin
spec:
  containers:
  - name: testpod
    image: busybox
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```
查看Pod的Service Account信息
```shell
$ kubectl  create -f anmin.yaml 
pod/testpod created
$ kubectl  describe pod  testpod  -n anmin
..........
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bskds (ro)
..........
Volumes:
  default-token-bskds:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-bskds
    Optional:    false
.........
```





在不指定ServiceAccount的情况下，当前 namespace 下面的 Pod 会默认使用 “default” 这个 ServiceAccount，对应的 Secret 会自动挂载到 Pod 的 /var/run/secrets/kubernetes.io/serviceaccount/ 目录中，我们可以在 Pod 里面获取到用于身份认证的信息。

```shell
$ kubectl  exec -it testpod -n anmin -- /bin/sh
/ # ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt     namespace  token
```



## 3. 使用自定义的ServiceAccount
创建一个Service Account
```shell
$ kubectl  create sa anmin -n anmin
serviceaccount/anmin created
$ kubectl  get sa -n anmin
NAME      SECRETS   AGE
anmin     1         20s
default   1         31m
$ kubectl   get secret -n anmin
NAME                  TYPE                                  DATA   AGE
anmin-token-nkb8b     kubernetes.io/service-account-token   3      28s
default-token-bskds   kubernetes.io/service-account-token   3      31m
```
Pod使用刚创建的ServiceAccount
```shell
apiVersion: v1
kind: Pod
metadata:
  name: testpod
  namespace: anmin
spec:
  containers:
  - name: testpod
    image: busybox
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
  serviceAccountName: anmin
 ```
 更新Pod
 ```shell
 $ kubectl   apply -f anmin.yaml 
pod/testpod created
$ kubectl  describe  pod  testpod -n anmin
.........
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from anmin-token-nkb8b (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  anmin-token-nkb8b:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  anmin-token-nkb8b
    Optional:    false
    ............
```
可以看到更新后的Pod已经使用了新创建的ServiceAccount服务账户。
## 4. ServiceAccount中添加Image pull secrets
我们也可以在Service Account中设置imagePullSecrets，然后就会自动为使用该 SA 的 Pod 注入 imagePullSecrets 信息。

创建kubernetes.io/dockerconfigjson类型的私有仓库镜像Secret
```shell
$ kubectl create secret docker-registry harbor --docker-server=http://192.168.166.229  --docker-username=admin   --docker-password=1234567  --docker-email=test@163.com   -n anmin
2secret/harbor created
```
将镜像仓库的Secret添加到ServiceAccount
```shell
$ kubectl  edit sa  anmin  -n anmin
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-06-16T16:09:54Z"
  name: anmin
  namespace: anmin
  resourceVersion: "37509823"
  selfLink: /api/v1/namespaces/anmin/serviceaccounts/anmin
  uid: c6bec7bb-808d-459f-86c8-6c78b48cb3ab
secrets:
- name: anmin-token-nkb8b
imagePullSecrets:
- name: harbor
```
查看ServiceAccount中Image pull secrets字段信息
```shell
$ kubectl  describe sa  anmin -n  anmin 
Name:                anmin
Namespace:           anmin
Labels:              <none>
Annotations:         <none>
Image pull secrets:  harbor
Mountable secrets:   anmin-token-nkb8b
Tokens:              anmin-token-nkb8b
Events:              <none>
```
使用ServiceAccount拉取私有镜像部署Pod
```shell
apiVersion: v1
kind: Pod
metadata:
  name: anmin2
  namespace: anmin
spec:
  containers:
  - name: anmin2
    image: 192.168.166.229/1an/node-exporter:latest
  serviceAccountName: anmin
```
更新Pod并查看状态
```shell
$ kubectl  apply -f harborsecret.yaml 
pod/anmin2 created
$ kubectl  get pod -n anmin
NAME         READY   STATUS             RESTARTS   AGE
anmin2       1/1     Running            0          20s
$ kubectl describe pod  anmin2 -n anmin
......
Volumes:
  anmin-token-nkb8b:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  anmin-token-nkb8b
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From                 Message
  ----    ------     ----  ----                 -------
  Normal  Pulling    8h    kubelet, k8s-node01  Pulling image "192.168.166.229/1an/node-exporter:latest"
  Normal  Pulled     8h    kubelet, k8s-node01  Successfully pulled image "192.168.166.229/1an/node-exporter:latest"
```
可以看到Pod已经成功从镜像仓库拉取镜像并正常运行。

参考资料：

https://time.geekbang.org/column/article/42154
https://www.qikqiak.com/k8s-book/docs/30.RBAC.html

------------------------------------------------------------------
关注公众号回复【k8s】获取视频教程及更多资料：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-52cdf2b971de1c50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


