# RBAC基于角色的权限控制--get started with a sample
基于角色的访问控制(RBAC)是通过用户角色来访问集群中的计算或者网络资源。  
RBAC使用rbac.authorization.k8s.io的API GROUP驱动权限模型，允许管理员动态配置访问权限策略。从1.8版本开始使用的API是rbac.authorization.k8s.io/v1  
使用RBAC需要集群的api-server启用RBAC认证。从1.6版本默认是开启的，通过以下命令查看是否开启。
```
$ cat /etc/kubernetes/manifests/kube-apiserver.yaml
...
    - --authorization-mode=Node,RBAC
...
```
# RBAC API 对象
RBAC API声明了本节将介绍的四种资源对象，通过这些资源对象，用户通过Kubectl能够访问集群内的其他资源，四种资源对象用于描述角色和权限、角色和用户的关系：
* Role
* ClusterRole
* RoleBinding
* ClusterRoleBinding 
   
Kubernetes有一个很基本的特性就是它的所有资源对象都是模型化的 API 对象，允许执行 CRUD(Create、Read、Update、Delete)操作(也就是我们常说的增、删、改、查操作)，比如下面的这下资源：

* Pods
* ConfigMaps
* Deployments
* Nodes
* Secrets
* Namespaces  
上面这些资源对象的可能存在的操作有：
* create
* get
* delete
* list
* update
* edit
* watch
* exec  
  
在更上层，这些资源和 API Group 进行关联，比如Pods属于 Core API Group，而Deployements属于 apps API Group，要在Kubernetes中进行RBAC的管理，除了上面的这些资源和操作以外，我们还需要另外的一些对象：

* Rule：规则，规则是一组属于不同 API Group 资源上的一组操作的集合
* Role 和 ClusterRole：角色和集群角色，这两个对象都包含上面的 Rules 元素，二者的区别在于，在 Role 中，定义的规则只适用于单个命名空间，也就是和 namespace 关联的，而 ClusterRole 是集群范围内的，因此定义的规则不受命名空间的约束。另外 Role 和 ClusterRole 在Kubernetes中都被定义为集群内部的 API 资源，和我们前面学习过的 Pod、ConfigMap 这些类似，都是我们集群的资源对象，所以同样的可以使用我们前面的kubectl相关的命令来进行操作
* Subject：主题，对应在集群中尝试操作的对象，集群中定义了3种类型的主题资源：   

  + User Account：用户，这是有外部独立服务进行管理的，管理员进行私钥的分配，用户可以使用 KeyStone或者 Goolge 帐号，甚至一个用户名和密码的文件列表也可以。对于用户的管理集群内部没有一个关联的资源对象，所以用户不能通过集群内部的 API 来进行管理
  + Group：组，这是用来关联多个账户的，集群中有一些默认创建的组，比如cluster-admin
  + Service Account：服务帐号，通过Kubernetes API 来管理的一些用户帐号，和 namespace 进行关联的，适用于集群内部运行的应用程序，需要通过 API 来完成权限认证，所以在集群内部进行权限操作，我们都需要使用到 ServiceAccount，这也是我们这节课的重点
* RoleBinding 和 ClusterRoleBinding：角色绑定和集群角色绑定，简单来说就是把声明的 Subject 和我们的 Role 进行绑定的过程(给某个用户绑定上操作的权限)，二者的区别也是作用范围的区别：RoleBinding 只会影响到当前 namespace 下面的资源操作权限，而 ClusterRoleBinding 会影响到所有的 namespace。

接下来我们来通过几个示例来演示下RBAC的配置方法。
# 创建一个只能访问某个namespace的用户
创建一个User Account，只能访问kube-system这个命名空间：
* username:wayne
* group:testcenter

## 创建用户凭证
kubernetes没有User Account的API对象，要创建一个用户账号利用管理员分配一个私钥就可以创建，参考官方文档中的方法[Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/),使用OpenSSL证书来创建一个User
* 给用户wayne创建一个私钥，命名成：wayne.key:
    ```
    openssl genrsa -out wayne.key 2048
    ```
* 使用我们刚刚创建的私钥创建一个证书签名请求文件：wayne.csr，-subj参数中指定用户名和组（CN表示用户名，O表示组）：
    ```
    openssl req -new -key wayne.key -out wayne.csr -subj "/CN=wayne/O=testcenter"
    ```
* 然后找Kubernetes集群的CA，我们使用的是kubeadm安装的集群，CA相关证书位于/etc/kubernetes/pki/目录下面，如果是二进制方式搭建的，应该在开始搭建集群时就已经制定好CA的目录，我们会利用该目录下面的ca.crt和ca.key两个文件来批准上面的证书请求
* 生成最终的证书文件，我们设置证书的有效期为500天：
    ```
    openssl x509 -req -in wayne.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out wayne.crt -days 500
    ```
* 查看当前文件夹下面是否生成一个证书文件：
    ```
    wayne.csr wayne.crt wayne.key
    ```
* 使用刚刚创建的证书文件和私钥文件创建新的凭证和上下文（Context）
    ```
    kubectl config set-credentials wayne --client-certificate=wayne.crt  --client-key=wayne.key
    ```
* 我们可以看到一个用户wayne创建了，然后为这个用户设置新的Context：
    ```
    kubectl config set-context wayne-context --cluster=kubernetes --namespace=kube-system --user=wayne
    ```
* 用户wayne创建成功了，现在我们使用当前的这个配置文件来操作kubectl命令的时候，应该会出现错误，因为我们还没有为该用户定义任何操作的权限呢：
    ```
    kubectl get pods --context=wayne-context
    Error from server (Forbidden): pods is forbidden: User "haimaxy" cannot list pods in the namespace "default"
    ```
## 创建角色
用户创建完成后，接下来就需要给用户添加操作权限，我们来定义一个YAML文件，创建一个允许用户操作Deployment、Pod、ReplicaSets的角色，如下定义（wayne-role.yaml）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wayne-role
  namespace: kube-system
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # 也可以使用['*']
```
然后创建这个Role：
```
kubectl create -f wayne-role.yaml
```
## 创建角色权限绑定
Role 创建完成了，但是很明显现在我们这个 Role 和我们的用户 wayne 还没有任何关系，对吧？这里我就需要创建一个RoleBinding对象，在 kube-system 这个命名空间下面将上面的 wayne-role 角色和用户 wayne 进行绑定:(wayne-rolebinding.yaml)
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: wayne-rolebinding
  namespace: kube-system
subjects:
- kind: User
  name: wayne
  apiGroup: ""
roleRef:
  kind: Role
  name: wayne-role
  apiGroup: ""  
```
上面的YAML文件中我们看到了subjects关键字，这里就是我们上面提到的用来尝试操作集群的对象，这里对应上面的User账号wayne，使用kubectl创建沙面的资源对象：
```
kubectl create -f wayne-rolebinding.yaml
```
## 测试
可以使用上面的wayne-context上下文来操作集群了：
```
kubectl get pods --context=wayne-context
```
我们可以看到我们使用kubectl的使用并没有指定 namespace 了，这是因为我们已经为该用户分配了权限了，如果我们在后面加上一个-n default试看看呢？
```
kubectl --context=wayne-context get pods --namespace=default
Error from server (Forbidden): pods is forbidden: User "wayne" cannot list pods in the namespace "default"
```
是符合我们预期的吧？因为该用户并没有 default 这个命名空间的操作权限
# 创建一个只能访问某个namespace的ServiceAccount
上面我们创建了一个只能访问某个命名空间下面的普通用户，subjects类型的主题资源还包括ServiceAccount，现在我们来创建一个集群内部的用户只能操作kube-system这个命名空间下面的pods和deployments，首先来创建一个ServiceAccount对象：
```
kubectl create sa wayne-sa -n kube-system
```
然后新建一个Role对象：（wayne-sa-role.yaml）
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wayne-sa-role
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] 
```
可以看到我们这里定义的角色没有创建、删除、更新 Pod 的权限，待会我们可以重点测试一下，创建该 Role 对象：
```
kubectl create -f wayne-sa-role.yaml
```
然后创建一个RoleBinding对象，将上面的wayne-sa和角色wayne-sa-role进行绑定（wayne-sa-rolebinding.yaml）
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: wayne-sa-rolebinding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: wayne-sa
  namespace: kube-system
roleRef:
  kind: Role
  name: wayne-sa-role
  apiGroup: rbac.authorization.k8s.io  
```
添加这个资源对象
```
kubectl create -f wayne-sa-rolebinding.yaml
```
然后我们怎么去验证这个 ServiceAccount 呢？我们前面的课程中是不是提到过一个 ServiceAccount 会生成一个 Secret 对象和它进行映射，这个 Secret 里面包含一个 token，我们可以利用这个 token 去登录 Dashboard，然后我们就可以在 Dashboard 中来验证我们的功能是否符合预期了：
```
kubectl get secret -n kube-system | grep wayne-sa
wayne-sa-token-w8gqx                             kubernetes.io/service-account-token   3      19h
kubectl get secret wayne-sa-token-w8gqx -o jsonpath={.data.token} -n kube-system |base64 -d
# 会生成一长串base64的字符串
```
使用这里的 token 去 Dashboard 页面进行登录：  
[imag](/images/rbac-default.png)
[imag](/images/rbac-kube-system.png)
我们可以看到上面的提示信息，这是因为我们登录进来后默认跳转到 default 命名空间，我们切换到 kube-system 命名空间下面就可以了：  
我们可以看到可以访问pod列表了，但是也会有一些其他额外的提示：events is forbidden: User “system:serviceaccount:kube-system:haimaxy-sa” cannot list events in the namespace “kube-system”，这是因为当前登录用只被授权了访问 pod 和 deployment 的权限，同样的，访问下deployment看看可以了吗？

同样的，你可以根据自己的需求来对访问用户的权限进行限制，可以自己通过 Role 定义更加细粒度的权限，也可以使用系统内置的一些权限……
# 创建一个可以访问集群的ServiceAccount
刚刚我们创建的wayne-sa这个 ServiceAccount 和一个 Role 角色进行绑定的，如果我们现在创建一个新的 ServiceAccount，需要他操作的权限作用于所有的 namespace，这个时候我们就需要使用到 ClusterRole 和 ClusterRoleBinding 这两种资源对象了。同样，首先新建一个 ServiceAcount 对象：(wayne-sa2.yaml)
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wayne-sa2
  namespace: kube-system  
```
创建sa
```
kubectl create -f wayne-sa2.yaml
```
然后创建一个ClusterRoleBinding对象（wayne-clusterrolebinding.yaml）
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: wayne-sa2-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: wayne-sa2
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io  
```
从上面我们可以看到我们没有为这个资源对象声明 namespace，因为这是一个 ClusterRoleBinding 资源对象，是作用于整个集群的，我们也没有单独新建一个 ClusterRole 对象，而是使用的 cluster-admin 这个对象，这是Kubernetes集群内置的 ClusterRole 对象，我们可以使用kubectl get clusterrole 和kubectl get clusterrolebinding查看系统内置的一些集群角色和集群角色绑定，这里我们使用的 cluster-admin 这个集群角色是拥有最高权限的集群角色，所以一般需要谨慎使用该集群角色。

创建上面集群角色绑定资源对象，创建完成后同样使用 ServiceAccount 对应的 token 去登录 Dashboard 验证下：
```
kubectl create -f wayne-clusterolebinding.yaml
kubectl get secret -n kube-system |grep wayne-sa2

kubectl get secret wayne-sa2-token-nxgqx -o jsonpath={.data.token} -n kube-system |base64 -d
# 会生成一串很长的base64后的字符串
```

我们在最开始接触到RBAC认证的时候，可能不太熟悉，特别是不知道应该怎么去编写rules规则，大家可以去分析系统自带的 clusterrole、clusterrolebinding 这些资源对象的编写方法，怎么分析？还是利用 kubectl 的 get、describe、 -o yaml 这些操作，所以kubectl最基本的用户一定要掌握好。

RBAC只是Kubernetes中安全认证的一种方式，当然也是现在最重要的一种方式。
# 参考
[Kubernetes RBAC 详解](https://blog.qikqiak.com/post/use-rbac-in-k8s/)  
[Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
