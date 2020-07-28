# ConfigMap
## 1. ConfigMap简介
在我们部署一些应用服务时，通常都会有一些配置文件，而在k8s中，我们如果在将这些配置文件写到代码应用程序中，需要修改配置的话，我们还得重新去修改代码，重新制作一个镜像，这样操作起来很麻烦。

还好kubernetes为我们提供了一个ConfigMap资源对象，它的主要作用就是为了让镜像和配置文件解耦，以便实现镜像的可移植性和可复用性，提供了向容器中注入配置信息的能力，不仅可以用来保存单个属性，还可以用来保存整个配置文件，我们只需要将配置文件以ConfigMap的方式挂载到应用Pod中，这样我们在更新配置信息的时候只需要更新这个ConfigMap对象即可。

ConfigMap使用场景：
- 通过环境变量的方式，直接传递给pod
- 通过在pod的命令行下运行的方式(启动命令中)
- 作为volume的方式挂载到pod内

## 2. 创建ConfigMap

### 2.1 通过目录创建

```bash
$ ls /app/web/configmap/kubectl/
ftp.properties
jdbc.properties

$ cat /app/web/configmap/kubectl/ftp.properties
FTP_ADDRESS=192.168.133.11
FTP_PORT=21
FTP_USERNAME=ftpuser
FTP_PASSWORD=ftpuser
FTP_BASEPATH=/
IMAGE_BASE_URL=http://test.test.com/image

$ cat /app/web/configmap/kubectl/jdbc.properties
#database configuration
connection.url=jdbc:mysql://10.1.133.111:3306/web
connection.username=testdata
#connection.password=abcde
connection.password=SN9+koFJWHmf
```
创建Configmap对象
```bash
$ kubectl create configmap web-config --from-file=/app/web/configmap/kubectl
```
`--from-file`：指定目录路径，该目录下的所有文件都会被用在ConfigMap里面创建一个键值对，键的名字就是文件名，值就是文件的内容。
查看Configmap信息
```bash
$ kubectl describe configmaps web-config
Name:         web-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
ftp.properties:
----
FTP_ADDRESS=192.168.133.11
FTP_PORT=21
FTP_USERNAME=ftpuser
FTP_PASSWORD=ftpuser
FTP_BASEPATH=/
IMAGE_BASE_URL=http://test.test.com/image

jdbc.properties:
----
#database configuration
connection.url=jdbc:mysql://10.1.133.111:3306/web
connection.username=testdata
#connection.password=abcde
connection.password=SN9+koFJWHmf

Events:  <none>
```
如果配置文件信息较多的话，describe的输出信息会显示不全，我们可以使用以下命令来查看完整内容：
```yaml
$ kubectl get configmaps web-config -o yaml
apiVersion: v1
data:
  ftp.properties: |
    FTP_ADDRESS=192.168.133.11
    FTP_PORT=21
    FTP_USERNAME=ftpuser
    FTP_PASSWORD=ftpuser
    FTP_BASEPATH=/
    IMAGE_BASE_URL=http://test.test.com/image
  jdbc.properties: |
    #database configuration
    connection.url=jdbc:mysql://10.1.133.111:3306/web
    connection.username=testdata
    #connection.password=abcde
    connection.password=SN9+koFJWHmf  
kind: ConfigMap
metadata:
  creationTimestamp: 2020-02-18T18:34:05Z
  name: web-config
  namespace: default
  resourceVersion: "407"
  selfLink: /api/v1/namespaces/default/configmaps/web-config
  uid: 680944725-d62e-10e5-8cd0-68f123db1985
```
### 2.2 使用文件创建
```yaml
$ kubectl create configmap web-config-2 --from-file=/app/web/configmap/kubectl/ftp.properties 
$ kubectl get configmaps web-config-2  -o yaml
apiVersion: v1
data:
  ftp.properties: |+
    FTP_ADDRESS=192.168.133.11
    FTP_PORT=21
    FTP_USERNAME=ftpuser
    FTP_PASSWORD=ftpuser
    FTP_BASEPATH=/
    IMAGE_BASE_URL=http://test.test.com/image

kind: ConfigMap
metadata:
  creationTimestamp: "2020-05-22T16:36:39Z"
  name: web-config-2
  namespace: default
  resourceVersion: "33386795"
  selfLink: /api/v1/namespaces/default/configmaps/web-config-2
  uid: cffef22c-be55-4de3-8835-79195de8ec7e
```
`--from-file`： 可以多次使用，指定多个文件
### 2.3 使用命令行创建
```yaml
$ kubectl create configmap web-config-3 --from-literal=FTP_ADDRESS=192.168.133.11  --from-literal=FTP_PORT=21  --from-literal=FTP_USERNAME=ftpuser   --from-literal=FTP_PASSWORD=ftpuser    --from-literal=FTP_BASEPATH=/  --from-literal=IMAGE_BASE_URL=http://test.test.com/image
configmap/web-config-3 created
$ kubectl get configmap web-config-3 -o yaml
apiVersion: v1
data:
  FTP_ADDRESS: 192.168.133.11
  FTP_BASEPATH: /
  FTP_PASSWORD: ftpuser
  FTP_PORT: "21"
  FTP_USERNAME: ftpuser
  IMAGE_BASE_URL: http://test.test.com/image
kind: ConfigMap
metadata:
  creationTimestamp: "2020-05-22T16:46:50Z"
  name: web-config-3
  namespace: default
  resourceVersion: "33387937"
  selfLink: /api/v1/namespaces/default/configmaps/web-config-3
  uid: d69afbaf-0ac4-4405-a4ea-351ad613bdb4
  ```
>  `--from-literal`：可以多次使用，指定多个值

## 3. 使用Configmp
### 3.1 使用ConfigMap来填充环境变量
编写Pod资源文件，使用上面名为web-config-3的configmap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testweb-pod
spec:
  containers:
    - name: testweb
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: FTP_ADDRESS
          valueFrom:
            configMapKeyRef:
              name: web-config-3
              key: FTP_ADDRESS  #configmap的key值
        - name: FTP_PORT
          valueFrom:
            configMapKeyRef:
              name: web-config-3
              key: FTP_PORT
      envFrom:
        - configMapRef:   
            name: web-config-3  #使用名为web-config-3的configmap对象
  ```
创建Pod并查看日志
```bash
$ kubectl  logs -f testweb-pod
--------
FTP_ADDRESS=192.168.133.11
IDLE_LLAMA_NGINX_INGRESS_DEFAULT_BACKEND_PORT_80_TCP=tcp://10.98.12.71:80
PLUNDERING_MARMOT_NGINX_INGRESS_CONTROLLER_PORT_80_TCP_ADDR=10.108.129.64
FTP_PORT=21
MY_RELEASE_TOMCAT_PORT_80_TCP_ADDR=10.101.28.152
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1

FTP_USERNAME=ftpuser
EXACERBATED_HEDGEHOG_NGINX_INGRESS_DEFAULT_BACKEND_SERVICE_PORT=80
PLUNDERING_MARMOT_NGINX_INGRESS_CONTROLLER_PORT_80_TCP=tcp://10.108.129.64:80
PLUNDERING_MARMOT_NGINX_INGRESS_CONTROLLER_PORT_443_TCP_ADDR=10.108.129.64
MYDB_MYSQL_SERVICE_PORT_MYSQL=3306
--------
```
### 3.2 使用ConfigMap设置命令行参数
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testweb2-pod
spec:
  containers:
    - name: testweb2
      image: busybox
      command: [ "/bin/sh", "-c", "echo $(FTP_ADDRESS) $(FTP_PORT)" ]
      env:
        - name: FTP_ADDRESS
          valueFrom:
            configMapKeyRef:
              name: web-config-3
              key: FTP_ADDRESS 
        - name: FTP_PORT
          valueFrom:
            configMapKeyRef:
              name: web-config-3
              key: FTP_PORT
```
查看Pod的输出信息
```bash
kubectl  logs -f testweb2-pod
192.168.133.11 21
```
### 3.3 通过数据卷使用ConfigMap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testweb3-pod
spec:
  volumes:
    - name: config-volume
      configMap:
        name: web-config-2
  containers:
    - name: testweb3
      image: busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/ftp.properties" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
   ```
 查看pod输出日志
 ```bash
 $ kubectl  logs -f testweb3-pod
FTP_ADDRESS=192.168.133.11
FTP_PORT=21
FTP_USERNAME=ftpuser
FTP_PASSWORD=ftpuser
FTP_BASEPATH=/
IMAGE_BASE_URL=http://test.test.com/image
```
>注意，当 ConfigMap 以数据卷的形式挂载进 Pod 的时，这时更新 ConfigMap（或删掉重建ConfigMap），Pod 内挂载的配置信息会热更新。这时可以增加一些监测配置文件变更的脚本，然后重新加载对应服务就可以实现应用的热更新。

cker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)

------------------------------------------------------------------
关注公众号回复【k8s】获取视频教程及更多资料：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-52cdf2b971de1c50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


