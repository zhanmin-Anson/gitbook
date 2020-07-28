# 使用Kubeadm部署集群
上篇文章主要介绍了[k8s(一) 基本概念与组件原理](https://www.jianshu.com/p/c231c7132fae)，下面我们来使用Kubeadm快速的部署一个 Kubernetes 集群，来理解 Kubernetes 组件的工作方式和架构。

#### 一.  kubeadm介绍

[kubeadm](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm/)是Kubernetes官方提供的用于快速安装 Kubernetes 集群的工具，它提供了 `kubeadm init `以及 `kubeadm join` 这两个命令作为快速创建 kubernetes 集群的最佳实践，只需将`kubeadm`,`kubelet`，`kubectl`安装到服务器，其他核心组件以容器化方式快速部署，不过目前kubeadm还处于beta状态，还不能用于生产环境。

#### 二. 环境准备
>系统版本： Centos7.2
内核版本：  3.10.0
 k8s    版本：   1.15.3
Docker版本：19.03

节点添加hosts信息
```
$ cat <<EOF >> /etc/hosts
172.16.1.100  k8s-master 
172.16.1.101  k8s-node01  
EOF
```
禁用Firewalld，Selinux，Swap
```
$ systemctl stop firewalld 
$ systemctl disable firewalld 
$ setenforce 0
$ sed -i "s/enforcing/disabled/g"  /etc/selinux/config
$ swapoff -a 
```
修改内核参数
```
$ cat << EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
$ modprobe br_netfilter #报错使用yum -y update 更新内核模块
$ sysctl -p /etc/sysctl.d/k8s.conf
```
安装[ipvs](https://segmentfault.com/a/1190000016333317)（负载均衡器，kube-proxy使用ipvs模式）

```
$ cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
$ chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```
安装[ipset](https://wiki.archlinux.org/index.php/Ipset) (iptables的扩展)
```
$ yum install ipvsadm ipset
```
同步服务器时间
```
$ yum -y install  ntp
$ ntpdate  ntp1.aliyun.com
```
#### 三. 安装docker
添加docker源并安装
```
$ yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum list docker-ce --showduplicates | sort -r
$ yum install docker-ce-19.03.1-3.el7

```
配置 Docker 镜像加速
```
$ mkdir /etc/docker
$ cat  << EOF  > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors" : [
    "https://ot2k4d59.mirror.aliyuncs.com/"
  ]
}
EOF
```
启动docker并设置开机自启
```
$ systemctl start docker
$ systemctl enable docker
```
####四. 安装Kubeadm
添加镜像源
```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
安装 kubeadm、kubelet、kubectl
```
$ yum -y install kubectl-1.15.3-0 kubeadm-1.15.3-0  kubelet-1.15.3-0
$ kubeadm version  #查看版本
$ kubeadm version: &version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:11:18Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
$ systemctl enable kubelet.service # 设置开机自启
```
####五. 初始化集群

**kubeadm初始化工作流程**

首先我们可以使用`kubeadm init`命令来进行初始化工作，其中kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes，比如检查内核版本是否是3.10以上，Cgroups 模块是否可用，Docker是否正确安装等，然后以Pod的形式来部署`kube-apiserver、kube-controller-manager、kube-scheduler`这些组件，最后则是部署`kube-proxy`和`DNS`这些插件。

如果我们需要使用一些自定义的配置，在Master节点可以导出默认的初始化文件进行修改。
```
$ kubeadm config print init-defaults > kubeadm.yaml
```
根据自己需求修改默认配置
```
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.1.100   #修改apiserverIP
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kubesphere
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: gcr.azk8s.cn/google_containers #修改镜像仓库地址
kind: ClusterConfiguration
kubernetesVersion: v1.15.3
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16  #Pod的IP网段，后面使用calico插件
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---  #添加以下，修改kube-proxy模式
apiVersion: kubeproxy.config.k8s.io/v1alpha1 
kind: KubeProxyConfiguration
mode: ipvs  
```
初始化集群

```
$ kubeadm init --config kubeadm.yaml
------
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of

ptions listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 172.16.1.100:6443 --token szu5t8.z6m03rxaamo8jzy1     --discovery-token-ca-cert-hash sha256:0455a39d0ff4cca1a9c947fa902ac635c09da5b4d7a30363e9376a9a2eb97a24
```
拷贝 kubeconfig 文件
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
####六. 节点加入集群

**kubeadm join工作流程**

在Master节点生成token后，然后在任意一台安装了 kubelet 和 kubeadm 的机器上执行 `kubeadm join`命令 即可加入到kubernetes集群中。
```
$ kubeadm join 172.16.1.100:6443 --token szu5t8.z6m03rxaamo8jzy1     --discovery-token-ca-cert-hash sha256:0455a39d0ff4cca1a9c947fa902ac635c09da5b4d7a30363e9376a9a2eb97a24
```
`上面的Key如果忘掉可以在master使用命令kubeadm token create --print-join-command重新获取。`

####七. 安装集群插件
#####1. 安装calico网络插件
```
$ wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
$ kubectl apply -f calico.yaml
```
查看Pod运行状态
```
$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5df986d44c-hpqrr   1/1     Running   0          67m
calico-node-nvhfh                          1/1     Running   0          63m
calico-node-vgft9                          1/1     Running   0          63m
coredns-cf8fb6d7f-q5kw6                    1/1     Running   0          2d19h
coredns-cf8fb6d7f-z92hh                    1/1     Running   0          2d19h
etcd-kubesphere                            1/1     Running   0          2d19h
kube-apiserver-kubesphere                  1/1     Running   0          2d19h
kube-controller-manager-kubesphere         1/1     Running   0          2d19h
kube-proxy-68n9f                           1/1     Running   0          2d19h
kube-proxy-6ht99                           1/1     Running   0          73m
kube-scheduler-kubesphere                  1/1     Running   0          2d19h
tiller-deploy-74cd79795-p26l5              1/1     Running   0          173m
```
 查看节点运行状态
 ```
$ kubectl  get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-node01   Ready    <none>   74m     v1.15.3
k8s-master   Ready    master   2d19h   v1.15.3
```
#####2. 安装Dashboard可视化插件
```
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
$ vim  kubernetes-dashboard.yaml #修改镜像名称
......
containers:
- args:
  - --auto-generate-certificates
  image: gcr.azk8s.cn/google_containers/kubernetes-dashboard-amd64:v1.10.1 # 修改镜像名称
  imagePullPolicy: IfNotPresent
......
selector:
  k8s-app: kubernetes-dashboard
type: NodePort  # 修改Service为NodePort类型
......
```
创建服务
```
$ kubectl apply -f kubernetes-dashboard.yaml
$ kubectl get pods -n kube-system -l k8s-app=kubernetes-dashboard
NAME                                  READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-fcfb4cbc-wr79d   1/1     Running   0          39s
$ kubectl get svc -n kube-system -l k8s-app=kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.96.168.30   <none>        443:31445/TCP   53s
```
创建一个具有所有权限的用户来登录Dashboard：
```
$ vim admin.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```
创建用户获取token
 ```
$ kubectl apply -f admin.yaml
$ kubectl get secret -n kube-system|grep admin-token
admin-token-4fjvq                                kubernetes.io/service-account-token   3      58s
$ kubectl get secret admin-token-4fjvq  -o jsonpath={.data.token} -n kube-system |base64 -d 
```

使用火狐浏览器访问Dashboard的NodePort端口  
`https://172.16.1.100:31445/`
![公众号：前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-9e4387411d32a1e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结：在这篇文章中，我们使用kubeadm工具来快速部署了k8s集群，理解了集群工作方式和架构，但因为 kubeadm 目前还是不能一键部署高可用的 Kubernetes 集群，如：Etcd、Apiserver等组件都应该是多节点集群，所以目前还是不能用于生产环境，关注公众号回复k8s部署获取二进制生产高可用部署文档。

---------------------------------------------------------------------------------------------------------------
上篇文章：[k8s一 | 基本概念与组件原理](https://www.jianshu.com/p/c231c7132fae)
系列文章：[深入理解Kuerneters](https://www.jianshu.com/nb/41327096)
参考资料：[从Docker到Kubernetes进阶-阳明](https://www.qikqiak.com/k8s-book/docs/18.YAML%20%E6%96%87%E4%BB%B6.html)
---------------------------------------------------------------------------------------------------------------
关注公众号回复【k8s】关键词获取视频教程及更多资料：
![前行技术圈](https://upload-images.jianshu.io/upload_images/9644303-bf8ad73af7f806b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




