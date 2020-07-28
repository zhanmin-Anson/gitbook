# RBAC
## 1. RBAC介绍
在Kubernetes 中所有资源对象都是通过 API 对象进行操作， 它们保存在 Etcd 里。而对Etcd的操作我们需要通过访问 kube-apiserver来实现，上面的Service Account其实就是APIServer的认证过程，而授权的机制则是通过RBAC：基于角色的访问控制实现的。

Role + RoleBinding + ServiceAccount 的权限分配方式是要重点掌握的内容。

RBAC的三个基本概念：

1. Role：角色，其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限；
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也就是在 Kubernetes 里定义的“用户”；
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。

## 2. Role与RoleBinding
现在我们通过实际操作来理解RBAC的工作机制
创建一个Service Account
```shell
$ kubectl  create sa  zhanmin-sa  -n kube-system
serviceaccount/zhanmin created
```

定义一个Role对象 zhanmin-sa-role.yaml
```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: zhanmin-sa-role
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
 ```
在上面的文件我们定义了被作用的命名空间为：kube-system，其中的rules 字段，就是它所定义的权限规则。其中规则定义的角色对Pod没有创建、删除、更新的权限。

其中的“被作用者”我们则是通过`RoleBinding`  对象来指定。

定义Rolebinding对象 zhanmin-sa-rolebinding.yaml 
```shell
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: zhanmin-sa-rolebinding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: zhanmin-sa
  namespace: kube-system
roleRef:
  kind: Role
  name: zhanmin-sa-role
  apiGroup: rbac.authorization.k8s.io
```
subjects 字段，即“被作用者”。它的类型是 User，即 Kubernetes 里的用户，也就是上文中的Service Account，这里我们定义被作用者用户为“zhanmin-sa”。

roleRef则是定义：RoleBinding 对象可以直接通过名字，来引用我们前面定义的 Role 对象，也就是“zhanmin-sa-role”，从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系。

所以Pod使用名为“zhanmin-sa”的ServiceAccount访问API Server时只能够做对Pod做get", "watch", "list"操作。这是因为“zhanmin-sa” 这个 ServiceAccount 的权限，已经被我们绑定了 Role 做了限制。

>注意：Role 和 RoleBinding 对象都是 Namespaced 对象（Namespaced Object），它们对权限的限制规则仅在它们自己的 Namespace 内有效，roleRef 也只能引用当前 Namespace 里的 Role 对象。

下面创建这些对象
```shell
$ kubectl   create -f zhanmin-sa-role.yaml 
role.rbac.authorization.k8s.io/zhanmin-sa-role created
$ kubectl  create -f  zhanmin-sa-rolebinding.yaml 
rolebinding.rbac.authorization.k8s.io/zhanmin-sa-rolebinding created
```
现在可以去之前部署的kubernetes-dashboard上验证权限
获取当前Service Account的Secret信息
```shell
$ kubectl  get secret  -n kube-system
zhanmin-sa-token-x6gxs                           kubernetes.io/service-account-token   3      136m
$ kubectl   get secret   zhanmin-sa-token-x6gxs  -o jsonpath={.data.token} -n kube-system |base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6InJCZFhYLTVRc2E4STlGVVN0VzEwWlc2M1VGMVF0ZDZFaFdJQlc3V2RLMzAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VeXN
```
现在可以去之前部署的kubernetes-dashboard上验证权限
命名空间修改为kube-system，因为上面我们已经说了Service Account只对当前的namespace有效。
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/9644303-b03f5cc0d56aa893?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/9644303-b8c5405e04a81a49?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，权限是符合我们上面的定义，只可以查看Pod和Deployments对象，查看其他资源比如SVC显示是没有数据的。后面我们可以根据自己的需求去查询API对象修改相应的权限规则。
## 3.  ClusterRole 和 ClusterRoleBinding 
上面的Role和RoleBinding只可以在他们自己的命名空间中有效，如果我们需要一个具有全部命名空间或者对节点有权限的角色时，就需要使用ClusterRole 和 ClusterRoleBinding 对象来做授权了。

ClusterRole 和 ClusterRoleBinding 这两个 API 对象的用法跟 Role 和 RoleBinding 几乎完全一样。不一样的是，它们的定义里，没有了 Namespace 字段，权限可以作用于整个集群。

创建ClusterRole集群角色  clusterrole.yaml 
```shell
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterrole-anmin
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
> 含义为：名为“clusterrole-anmin”的集群角色可以对集群所有命名空间的Pod进行“GET、Watch、List” 操作。

在Role 或者 ClusterRole 里面，如果要赋予用户 example-user 所有权限，那你就可以给它指定一个 verbs 字段的全集，如下所示：
```shell
verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
创建 ClusterRoleBinding集群角色绑定  ClusterRoleBinding.yaml
```shell
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: user-anmin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: clusterrole-anmin
  apiGroup: rbac.authorization.k8s.io
```
>含义为： subjects字段定义被作用者用户为“user-anmin”，roleRef字段定义：绑定名为“clusterrole-anmin”集群角色。

在 Kubernetes 中已经内置了很多个为系统保留的 ClusterRole，它们的名字都以 system: 开头。你可以通过 `kubectl get clusterroles `查看到它们。

查看集群角色
```shell
$ kubectl get clusterroles 
NAME                                                                   AGE
admin                                                                  242d
calico-kube-controllers                                                211d
calico-node                                                            211d
cluster-admin                                                          242d
cluster-regular                                                        217d
edit                                                                   242d
......
```
查看角色的权限
```shell
$ kubectl describe clusterrole edit 
Name:         edit
Labels:       kubernetes.io/bootstrapping=rbac-defaults
              rbac.authorization.k8s.io/aggregate-to-admin=true
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                Non-Resource URLs  Resource Names  Verbs
  ---------                                -----------------  --------------  -----
  configmaps                               []                 []              [create delete deletecollection patch update get list watch]
  endpoints                                []                 []              [create delete deletecollection patch update get list watch]
  persistentvolumeclaims                   []                 []              [create delete deletecollection patch update get list watch]
  pods                                     []                 []              [create delete deletecollection patch update get list watch]
  replicationcontrollers/scale             []                 []              [create delete deletecollection patch update get list watch]
......
```
## 4. Group用户组
Kubernetes 还拥有“用户组”（Group）的概念，也就是一组“用户”的意思。如果你为 Kubernetes 配置了外部认证服务的话，这个“用户组”的概念就会由外部认证服务提供。  

ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：
```
system:serviceaccount:<Namespace名字>:<ServiceAccount名字>
```
对应的内置“用户组”的名字，就是：
```
system:serviceaccounts:<Namespace名字>
```

比如，现在我们可以在 RoleBinding 里定义如下的 subjects：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```
这就意味着这个 Role 的权限规则，作用于 mynamespace 里的所有 ServiceAccount。这就用到了“用户组”的概念。而下面这个例子：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```
就意味着这个 Role 的权限规则，作用于整个系统里的所有 ServiceAccount。

总结：通过上面的实践，我们了解了在kubernetes中用户分为User Accounts和 Service Accounts，在我们平常的使用中会经常使用ServiceAccount。而对于权限的控制，我们需要先创建角色（Role），其实就是一组权限规则列表。然后我们分配这些权限的方式，就是通过创建 RoleBinding 对象，将被作用者（subject）Service Account和权限列表Role进行绑定，也就是Role + RoleBinding + ServiceAccount来实现。另外ClusterRole 和 ClusterRoleBinding，则是 Kubernetes 集群级别的 Role 和 RoleBinding，它们的作用范围不受 Namespace 限制。

参考资料：

https://time.geekbang.org/column/article/42154
https://www.qikqiak.com/k8s-book/docs/30.RBAC.html



------------------------------------------------------------------
关注公众号回复【k8s】获取视频教程及更多资料：
![image.png](https://upload-images.jianshu.io/upload_images/9644303-52cdf2b971de1c50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

