# Secret
## 1. Secret简介
 一般情况下ConfigMap 是用来存储一些非安全的配置信息，因为ConfigMap是明文存储的，面对敏感信息时，我们就需要使用k8s的另一个对象Secret。

Secret用来保存敏感信息，例如密码、OAuth 令牌和 ssh key 等等，将这些信息放在 Secret 中比放在 Pod 的定义中或者 Docker 镜像中要更加安全和灵活。

Secret 主要使用的有以下三种类型：

- Opaque：base64 编码格式的 Secret，用来存储密码、密钥等；但数据也可以通过base64 –decode解码得到原始数据，所有加密性很弱。
- kubernetes.io/dockerconfigjson：用来存储私有docker registry的认证信息。
- kubernetes.io/service-account-token：用于被 ServiceAccount ServiceAccount 创建时 Kubernetes 会默认创建一个对应的 Secret 对象。Pod 如果使用了 ServiceAccount，对应的 Secret 会自动挂载到 Pod 目录 /run/secrets/kubernetes.io/serviceaccount 中。
bootstrap.kubernetes.io/token：用于节点接入集群的校验的 Secret

Secret对比ConfigMap

相同点
- key/value的形式
- 属于某个特定的命名空间
- 可以导出到环境变量
- 可以通过目录/文件形式挂载
- 通过 volume 挂载的配置信息均可热更新

不同点
- Secret 可以被 ServerAccount 关联
- Secret 可以存储 docker register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像
- Secret 支持 Base64 加密
- Secret 分为 kubernetes.io/service-account-token、kubernetes.io/dockerconfigjson、Opaque 三种类型，而 Configmap 不区分类型

## 2. Opaque Secret
### 2.1 手动创建Secret
```bash
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "adminpass" | base64
YWRtaW5wYXNz
```
>Opaque 类型的数据是一个 map 类型，要求 value 必须是 base64 编码格式。

使用上面加密的编码编写Yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW5wYXNz
```
创建Secret
```bash
$ kubectl  create -f secret-demo.yaml 
secret/mysecret created
$ kubectl  get  secret
NAME                                             TYPE                                  DATA   AGE
default-token-g7sqf                              kubernetes.io/service-account-token   3      219d
exacerbated-hedgehog-nginx-ingress-token-zrwpn   kubernetes.io/service-account-token   3      174d
idle-llama-nginx-ingress-token-r2j5k             kubernetes.io/service-account-token   3      174d
istio.default                                    istio.io/key-and-cert                 3      185d
mydb-mysql                                       Opaque                                2      181d
mysecret                                         Opaque                                2      6m13s
```
查看Secret信息 
```bash
kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  9 bytes
username:  5 bytes
```
查看Secret详细信息
```yaml
apiVersion: v1
data:
  password: YWRtaW5wYXNz
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2020-05-25T14:50:01Z"
  name: mysecret
  namespace: default
  resourceVersion: "33864313"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: cf560ef9-eb67-4b3d-ac17-ed930e6799ce
type: Opaque
```
### 2.2 使用Secret作为环境变量
编写Yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret1-pod
spec:
  containers:
  - name: secret1
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
```
>`secretKeyRef` ：和前面Configmap定义相同，获取Secret的key值

创建Pod后查看日志输出
```bash
$  kubectl  create -f testpod.yaml 
pod/secret1-pod created
$ kubectl   logs -f secret1-pod | grep  USERNAME
USERNAME=admin
$ kubectl   logs -f secret1-pod | grep  PASSWORD
PASSWORD=adminpass
```
### 2.3 通过数据卷使用Secret
将Sercret挂载到Pod的/etc/secret路径下
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret2-pod
spec:
  containers:
  - name: secret2
    image: busybox
    command: ["/bin/sh", "-c", "ls /etc/secrets"]
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
  volumes:
  - name: secrets
    secret:
     secretName: mysecret
```
创建Pod并查看日志输出
```bash
$ kubectl  create -f testpod2.yaml 
 pod/secret2-pod created
$ kubectl  logs -f secret2-pod
password
username
```
可以看到 Secret 把两个 key 以两个对应的文件挂载到了Pod中/etc/secret路径下，如果挂载到指定的文件上面，可以使用上面Configmap的数据卷挂载方式。
## 3. kubernetes.io/dockerconfigjson
### 3.1 创建Secret对象
这个类型的Secret对象用来创建建镜像仓库认证信息，如下：
```bash
$ kubectl create secret docker-registry harbor --docker-server=http://192.168.166.229  --docker-username=admin   --docker-password=harbor123 --docker-email=test@163.com
secret/harbor created
```
查看创建的secret，注意Sercret对应的TYPE类型
```bash
$ kubectl get secret
NAME                                             TYPE                                  DATA   AGE
default-token-g7sqf                              kubernetes.io/service-account-token   3      233d
exacerbated-hedgehog-nginx-ingress-token-zrwpn   kubernetes.io/service-account-token   3      188d
harbor                                           kubernetes.io/dockerconfigjson        1      66s
```
### 3.2  查看Secret信息
```yaml
$ kubectl get secret  harbor  -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwOi8vMTkyLjE2OC4xNjYuMjI5Ijp7InVzZXJuYW1lIjoiYWRtaW4iLCJwYXNzd29yZCI6IkhhcmJvckAxYW4iLCJlbWFpbCI6InRlc3RAMTYzLmNvbSIsImF1dGgiOiJZV1J0YVc0NlNHRnlZbTl5UURGaGJnPT0ifX19
kind: Secret
metadata:
  creationTimestamp: "2020-06-08T14:20:41Z"
  name: harbor
  namespace: default
  resourceVersion: "36160653"
  selfLink: /api/v1/namespaces/default/secrets/harbor
  uid: 69f79c70-245a-4a91-b846-878a026e7ba6
type: kubernetes.io/dockerconfigjson
```
可以看到dockerconfigjson信息已经被加密了，我们可以使用base64解码查看
```bash
$ echo eyJhdXRocyI6eyJodHRwOi8vMTkyLjE2OC4xNjYuMjI5Ijp7InVzZXJuYW1lIjoiYWRtaW4iLCJwYXNzd29yZCI6IkhhcmJvckAxYW4iLCJlbWFpbCI6InRlc3RAMTYzLmNvbSIsImF1dGgiOiJZV1J0YVc0NlNHRnlZbTl5UURGaGJnPT0ifX19  |  base64 -d 
{"auths":{"http://192.168.166.229":{"username":"admin","password":"harbor123 ","email":"test@163.com","auth":"YWRtaW46SGFyYm9yQDFhbg=="}}}
```

### 3.3 Pod使用Secret对象
使用名为harbor的Secret拉取镜像：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: harbortest
spec:
  containers:
  - name: harbor
    image: 192.168.166.229/test/node-exporter:latest
  imagePullSecrets:
  - name: harbor
 ```
 查看Pod状态
 ```bash
$ kubectl get pod 
NAME                                                              READY   STATUS             RESTARTS   AGE
harbortest                                                        1/1     Running            0          2m26s
 ```
 查看Pod创建过程，镜像是否符合定义
 ```bash
 $ kubectl  describe  pod  harbortest   
Events:
  Type    Reason     Age    From                 Message
  ----    ------     ----   ----                 -------
  Normal  Scheduled  3m2s   default-scheduler    Successfully assigned default/harbortest to kubesphere
  Normal  Pulling    2m57s  kubelet, kubesphere  Pulling image "192.168.166.229/test/node-exporter:latest"
  Normal  Pulled     2m57s  kubelet, kubesphere  Successfully pulled image "192.168.166.229/test/node-exporter:latest"
  Normal  Created    2m51s  kubelet, kubesphere  Created container harbor
  Normal  Started    2m51s  kubelet, kubesphere  Started container harbor
 ```
 ## 4.kubernetes.io/service-account-token
`kubernetes.io/service-account-token`类型的Secret用于被 ServiceAccount （服务账户）引用。

Service Account：为Pod中的进程和外部用户提供身份信息。ServiceAccout 创建时 Kubernetes 会默认创建对应的 Secret。

集群创建时会创建一个默认的Service Account，
```bash
$ kubectl  get sa
NAME                                 SECRETS   AGE
default                              1         233d
$ kubectl     describe sa default 
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-g7sqf
Tokens:              default-token-g7sqf
Events:              <none>
```
可以看到名为“default”的Service Account对应的service-account-token
为“default-token-g7sqf”，这是因为ServiceAccout 创建时 Kubernetes 会默认创建对应的 Secret。

然后我们查看之前创建pod的详细信息
```yaml
$ kubectl  describe pod web-0
.........................
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-g7sqf (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-g7sqf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-g7sqf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```
可以看到创建的pod使用了名为“default-token-g7sqf”对应的service-account-token，这是因为我们使用`kubectl`指令请求apiserver创建的时候，使用的Service Account服务账户为`default`,而名为”default“ 的Service Account对应的service-account-token就是“default-token-g7sqf”。对应的 Secret 对象信息会通过 Volume 挂载到容器的/var/run/secrets/kubernetes.io/serviceaccount目录中。

关于Service Account 对象我会在下篇文章中详细解析，欢迎关注。

参考资料：[从Docker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)

------------------------------------------------------------------
关注公众号回复【k8s】获取视频教程及更多资料：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-52cdf2b971de1c50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


