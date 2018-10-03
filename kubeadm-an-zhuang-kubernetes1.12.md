# Kubeadm 安装kubernetes 1.12

kubeadm是官方推荐的kubernetes的安装教程，由于国内搭建过程中拉取google的镜像有网络问题，所以搭建之前首先需要解决网络问题，可以通过阿里云的香港节点通过SS做流量代理。
# 准备工作
## 网络环境准备
## centos7下使用ss代理,开启docker的sock5代理
### 1.安装客户端
```
yum -y install epel-release
yum -y install python-pip
pip install shadowsocks
```
新建客户端配置文件
```
mkdir /etc/shadowsocks
vi /etc/shadowsocks/shadowsocks.json
```
添加配置信息如下
```
{
    "server":"1.1.1.1",
    "server_port":1035,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"password",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}
```
配置开机启动，新建脚本文件/etc/systemd/system/shadowsocks.service
```
[Unit]
Description=Shadowsocks
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks/shadowsocks.json
[Install]
WantedBy=multi-user.target
```
启动SS客户端
```
systemctl enable shadowsocks.service
systemctl start shadowsocks.service
systemctl status shadowsocks.service
```
验证SS客户端是否正常运行
```
curl --socks5 127.0.0.1:1080 http://httpbin.org/ip
```
如果SS客户端正常运行，则结果如下
```
{
  "origin": "x.x.x.x"       #你的Shadowsock服务器IP
}
```
### 安装配置Privoxy
SS使用socks5协议通信，常见wget，curl协议走http协议，需要一种将Http代理桥接成socks5的方案，Privoxy就是做这个转换的工具。
安装privoxy
```
yum -y install privoxy
systemctl enable privoxy
systemctl start privoxy
systemctl status privoxy
```
修改配置文件/etc/privoxy/config
```
确保如下内容没有被注释掉
listen-address 127.0.0.1:8118 # 8118 是默认端口，不用改
forward-socks5t / 127.0.0.1:1080 . #转发到本地端口，注意最后有个点
```
设置http/https代理，在/etc/profile文件中添加如下内容
```
PROXY_HOST=127.0.0.1
export all_proxy=http://$PROXY_HOST:8118
export ftp_proxy=http://$PROXY_HOST:8118
export http_proxy=http://$PROXY_HOST:8118
export https_proxy=http://$PROXY_HOST:8118
export no_proxy=localhost,172.16.0.0/16,192.168.0.0/16.,127.0.0.1,10.10.0.0/16
```
使环境变量生效
```
source /etc/profile
```
测试
```
curl www.google.com
```
取消使用代理
```
while read var; do unset $var; done < <(env | grep -i proxy | awk -F= '{print $1}')
```
### docker使用socks5,由于当前docker没有安装，可在安装docker后设置代理
验证一下代理端口是否生效
```
[root@cloud4ourself-kcluster1 ~]# grep socks5 /etc/privoxy/config |grep -v ^# 
forward-socks5 / 127.0.0.1:1080 . 
[root@cloud4ourself-kcluster1 ~]# systemctl start privoxy 
[root@cloud4ourself-kcluster1 ~]# export http_proxy=http://127.0.0.1:8118 [root@cloud4ourself-kcluster1 privoxy]# ss -antp |grep 1080 LISTEN 0 128 127.0.0.1:1080 *:* users:(("sslocal",pid=19727,fd=4)) 
[root@cloud4ourself-kcluster1 privoxy]# ss -antp |grep 8118 LISTEN 0 128 127.0.0.1:8118 *:* users:(("privoxy",pid=30250,fd=4))
```
docker的配置文件vim /usr/lib/systemd/system/docker.service
[Service]下增加
```
Environment=HTTP_PROXY=http://127.0.0.1:8118/ 
Environment=HTTPS_PROXY=http://127.0.0.1:8118/ 
Environment=NO_PROXY=localhost,127.0.0.1,m1empwb1.mirror.aliyuncs.com,docker.io,
registry.cn-hangzhou.aliyuncs.com
```
```
[root@cloud4ourself-kcluster1 system]# systemctl daemon-reload
[root@cloud4ourself-kcluster1 system]# systemctl show docker |grep 127.0.0.1 
Environment=GOTRACEBACK=crash HTTP_PROXY=http://127.0.0.1:8118/ HTTPS_PROXY=http://127.0.0.1:8118/ NO_PROXY=localhost,127.0.0.1,m1empwb1.mirror.aliyuncs.com,docker.io,registry.cn-hangzhou.aliyuncs.com 
[root@cloud4ourself-kcluster1 system]# systemctl restart docker
```
测试
```
[root@node1 ~]# ss -antp |grep EST |egrep '1080|8118'
ESTAB      0      0      127.0.0.1:50028              127.0.0.1:1080                users:(("privoxy",pid=19768,fd=5))
ESTAB      0      0      127.0.0.1:1080               127.0.0.1:50028               users:(("sslocal",pid=19573,fd=8))
ESTAB      0      0      127.0.0.1:51204              127.0.0.1:8118                users:(("kube-apiserver",pid=22225,fd=98))
ESTAB      0      0      127.0.0.1:8118               127.0.0.1:51204               users:(("privoxy",pid=19768,fd=3))
```
## 系统设置
准备的三台Centos7.3主机如下：
```
[root@node1 ~]# cat /etc/hosts
10.10.4.12  node1
10.10.4.13  node2
10.10.4.14  node3
```
各节点之间通信需要用到不同端口号，需要设置防火墙过滤规则，具体规则在[Check Required Ports](https://kubernetes.io/docs/setup/independent/install-kubeadm/#check-required-ports)中。由于我这边是内网环境，为简单起见，我直接关闭防火墙。
```
systemctl stop firewalld
systemctl disable firewalld
```
禁用SELinux
```
setenforce 0
vi /etc/selinux/config
SELINUX=disabled
```
iptables被绕过导致路由不正确，需要设置net.bridge.bridge-nf-call-iptables为1
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```
## 安装Docker
添加docker-ce的yum源
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
查看当前的Docker版本
```
yum list docker-ce.x86_64  --showduplicates |sort -r
docker-ce.x86_64            18.06.1.ce-3.el7                   docker-ce-stable 
docker-ce.x86_64            18.06.0.ce-3.el7                   docker-ce-stable 
docker-ce.x86_64            18.03.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            18.03.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.12.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.12.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.09.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.09.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.2.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.3.ce-1.el7                   docker-ce-stable 
docker-ce.x86_64            17.03.2.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.2.ce-1.el7.centos            @docker-ce-stable
docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
```
kubernetes对docker17.03做过验证，所以我们安装docker17.03版本
```
yum makecache fast

yum install -y --setopt=obsoletes=0 \
  docker-ce-17.03.2.ce-1.el7.centos \
  docker-ce-selinux-17.03.2.ce-1.el7.centos

systemctl start docker
systemctl enable docker
```
Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信，在各个Docker节点执行下面的命令：
```
iptables -P FORWARD ACCEPT
```
可在docker的systemd unit文件中以ExecStartPost加入上面的命令：
```
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
systemctl daemon-reload
systemctl restart docker
```
# 安装Kubeadm
## 在每个节点上安装kubeadm和kubelet
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```
执行安装以及执行日志
```
yum makecache fast
yum install -y kubelet kubeadm kubectl

... 
Installed:
  kubeadm.x86_64 0:1.12.0-0    kubectl.x86_64 0:1.12.0-0    kubelet.x86_64 0:1.12.0-0

Dependency Installed:
  cri-tools.x86_64 0:1.12.0-0  kubernetes-cni.x86_64 0:0.6.0-0  socat.x86_64 0:1.7.3.2-2.el7
```
kubernetes要求关闭系统的swap,如果不关闭，默认配置下kubelet将无法启动。
关闭系统的Swap方法如下
```
swapoff -a
修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭。 swappiness参数调整，修改/etc/sysctl.d/k8s.conf添加下面一行：
vm.swappiness=0
执行sysctl -p /etc/sysctl.d/k8s.conf使修改生效。
```
# 创建Master节点
## kubeadm初始化集群
设置docker和kubelet开机启动，在三个节点上
```
systemctl enable docker
systemctl enable kubelet
```
接下来使用kubeadm初始化集群，选择node1作为Master Node，在node1上执行下面的命令：
```
[root@localhost ~]# kubeadm init   --kubernetes-version=v1.12.0   --pod-network-cidr=10.244.0.0/16   --apiserver-advertise-address=10.10.4.12
[init] using Kubernetes version: v1.12.0
[preflight] running pre-flight checks
	[WARNING HTTPProxyCIDR]: connection to "10.96.0.0/12" uses proxy "http://127.0.0.1:8118". This may lead to malfunctional cluster setup. Make sure that Pod and Services IP ranges specified correctly as exceptions in proxy configuration
	[WARNING HTTPProxyCIDR]: connection to "10.244.0.0/16" uses proxy "http://127.0.0.1:8118". This may lead to malfunctional cluster setup. Make sure that Pod and Services IP ranges specified correctly as exceptions in proxy configuration
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [node1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.10.4.12]
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [node1 localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [node1 localhost] and IPs [10.10.4.12 127.0.0.1 ::1]
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[certificates] Generated sa key and public key.
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 24.503142 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.12" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node node1 as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node node1 as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "node1" as an annotation
[bootstraptoken] using token: 0be9k5.l1hm3oxdprqpzwqc
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.10.4.12:6443 --token 0be9k5.l1hm3oxdprqpzwqc --discovery-token-ca-cert-hash sha256:01d340f9b0e00b68afee3e1224f7c9a0fb37f0008d1374739b8f6debaa40bc12

```
因为我们选择flannel作为Pod网络插件，所以上面的命令指定–pod-network-cidr=10.244.0.0/16。
查看一下集群状态。
```
[root@node1 ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"} 
```
确认个组件都处于healthy状态。

集群初始化如果遇到问题，可以使用下面的命令进行清理：
```
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```
## 安装flannel网络插件
```
[root@node1 k8s]# wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml

[root@node1 k8s]# kubectl apply -f kube-flannel.yml 
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created
```
使用kubectl get pod --all-namespaces -o wide确保所有的Pod都处于Running状态
```
[root@node1 ~]# kubectl get pod --all-namespaces
NAMESPACE       NAME                                             READY   STATUS    RESTARTS   AGE
default         curl-5cc7b478b6-t8bp9                            1/1     Running   1          22h
ingress-nginx   nginx-ingress-controller-7b44476955-r5rdf        1/1     Running   0          19h
ingress-nginx   nginx-ingress-default-backend-7b98d4dc6c-nc5cv   1/1     Running   0          19h
kube-system     coredns-576cbf47c7-8khz2                         1/1     Running   4          22h
kube-system     coredns-576cbf47c7-vbjwh                         1/1     Running   4          23h
kube-system     etcd-node1                                       1/1     Running   0          23h
kube-system     heapster-684777c4cb-nzdsx                        1/1     Running   0          14h
kube-system     heapster-heapster-657c6bfb8d-xvd28               2/2     Running   0          14h
kube-system     kube-apiserver-node1                             1/1     Running   0          23h
kube-system     kube-controller-manager-node1                    1/1     Running   0          23h
kube-system     kube-flannel-ds-gcs87                            1/1     Running   0          21h
kube-system     kube-flannel-ds-tsn6l                            1/1     Running   0          22h
kube-system     kube-flannel-ds-x982c                            1/1     Running   0          22h
kube-system     kube-proxy-l8vt5                                 1/1     Running   0          23h
kube-system     kube-proxy-mt69g                                 1/1     Running   0          21h
kube-system     kube-proxy-pzxct                                 1/1     Running   0          22h
kube-system     kube-scheduler-node1                             1/1     Running   0          23h
kube-system     kubernetes-dashboard-5746dd4544-qnkgj            1/1     Running   0          15h
kube-system     monitoring-grafana-56b668bccf-dql9w              1/1     Running   0          14h
kube-system     monitoring-influxdb-5c5bf4949d-kc5vd             1/1     Running   0          14h
kube-system     tiller-deploy-6f6fd74b68-88rzm                   1/1     Running   0          20h
```
## Master node 参与工作负载
使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。这是因为master被打上了污点。
```
kubectl describe node node1 | grep Taint
Taints:             node-role.kubernetes.io/master:NoSchedule
```
因为这里搭建的是测试环境，去掉这个污点使node1参与工作负载：
```
kubectl taint nodes node1 node-role.kubernetes.io/master-
node "node1" untainted
```
## 测试DNS
```
kubectl run curl --image=radial/busyboxplus:curl -i --tty
If you don't see a command prompt, try pressing enter.
[ root@curl-2716574283-xr8zd:/ ]$
nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```
进入后执行nslookup kubernetes.default确认解析正常
# 添加工作节点
将4.13，4.14添加到集群中，首先需要在节点中关闭swap.
```
[root@localhost ~]# kubeadm join 10.10.4.12:6443 --token 0be9k5.l1hm3oxdprqpzwqc --discovery-token-ca-cert-hash sha256:01d340f9b0e00b68afee3e1224f7c9a0fb37f0008d1374739b8f6debaa40bc12
[preflight] running pre-flight checks
	[WARNING RequiredIPVSKernelModulesAvailable]: the IPVS proxier will not be used, because the following required kernel modules are not loaded: [ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh] or no builtin kernel ipvs support: map[ip_vs:{} ip_vs_rr:{} ip_vs_wrr:{} ip_vs_sh:{} nf_conntrack_ipv4:{}]
you can solve this problem with following methods:
 1. Run 'modprobe -- ' to load missing kernel modules;
2. Provide the missing builtin kernel ipvs support

	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
[discovery] Trying to connect to API Server "10.10.4.12:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://10.10.4.12:6443"
[discovery] Requesting info from "https://10.10.4.12:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "10.10.4.12:6443"
[discovery] Successfully established connection with API Server "10.10.4.12:6443"
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.12" ConfigMap in the kube-system namespace
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[preflight] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "node2" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```
Node节点加入很顺利，但是在master节点上执行kubectl get nodes,会发现新加入的节点是NotReady状态。
问题排查，查看pod状态，会发现flannel的pod一直是pending状态，查看Event事件显示0/3 nodes are available: 1 node(s) had taints that the pod didn't tolerate。这是因为工作节点添加了脏点限制，如下：
```
[root@node1 k8s]# kubectl get po -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-576cbf47c7-8khz2        1/1     Running   4          81m
coredns-576cbf47c7-vbjwh        1/1     Running   4          134m
etcd-node1                      1/1     Running   0          134m
kube-apiserver-node1            1/1     Running   0          134m
kube-controller-manager-node1   1/1     Running   0          133m
kube-flannel-ds-gcs87           0/1     Pending   0          4m26s
kube-flannel-ds-tsn6l           1/1     Running   0          77m

[root@node1 k8s]# kubectl describe po kube-flannel-ds-gcs87 -n kube-system
Events:
  Type     Reason            Age                     From               Message
  ----     ------            ----                    ----               -------
  Warning  FailedScheduling  3m32s (x25 over 4m42s)  default-scheduler  0/3 nodes are available: 1 node(s) had taints that the pod didn't tolerate, 2 node(s) didn't match node selector.

[root@node1 k8s]# kubectl describe nodes node3 | grep Taint
Taints:             node.kubernetes.io/not-ready:NoSchedule

去掉污点：
[root@node1 k8s]# kubectl describe nodes node3 | grep Taint
Taints:             <none>
```
## 如何移除节点
如果需要从集群中移除node2这个Node执行下面的命令：

在master节点上执行：
```
kubectl drain node2 --delete-local-data --force --ignore-daemonsets
kubectl delete node node2
```
在node2上执行：
```
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```
# Kubernetes常用组件部署
## Helm安装
Helm由客户端命helm令行工具和服务端tiller组成，Helm的安装十分简单。 下载helm命令行工具到master节点node1的/usr/local/bin下，这里下载的2.11.0版本：
```
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz
tar -zxvf helm-v2.11.0-linux-amd64.tar.gz
cd linux-amd64/
cp helm /usr/local/bin/
```
为了安装服务端tiller，还需要在这台机器上配置好kubectl工具和kubeconfig文件，确保kubectl工具可以在这台机器上访问apiserver且正常使用。 这里的node1节点以及配置好了kubectl。

因为Kubernetes APIServer开启了RBAC访问控制，所以需要创建tiller使用的service account: tiller并分配合适的角色给它。 详细内容可以查看helm文档中的[Role-based Access Control](https://docs.helm.sh/using_helm/#role-based-access-control)。 这里简单起见直接分配cluster-admin这个集群内置的ClusterRole给它。创建rbac-config.yaml文件：
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
```
kubectl create -f rbac-config.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```
接下来使用helm部署tiller:
```
helm init --service-account tiller --skip-refresh
Creating /root/.helm
Creating /root/.helm/repository
Creating /root/.helm/repository/cache
Creating /root/.helm/repository/local
Creating /root/.helm/plugins
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```
tiller默认被部署在k8s集群中的kube-system这个namespace下：
```
kubectl get pod -n kube-system -l app=helm
NAME                            READY     STATUS    RESTARTS   AGE
tiller-deploy-759cb9df9-fdg58   1/1       Running   0          2m
```
```
helm version
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
```
注意由于某些原因需要网络可以访问gcr.io和kubernetes-charts.storage.googleapis.com，如果无法访问可以通过helm init --service-account tiller --tiller-image <your-docker-registry>/tiller:2.11.0 --skip-refresh使用私有镜像仓库中的tiller镜像
## 使用Hlem部署Nginx Ingress
为了便于将集群中的服务暴露到集群外部，从集群外部访问，接下来使用Helm将Nginx Ingress部署到Kubernetes上。
ingress-nginx.yaml：
```
controller:
  service:
    externalIPs:
      - 10.10.4.12

[root@node1 k8s]# vi ingress-nginx.yaml
[root@node1 k8s]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 
```
```
curl http://10.10.4.12
default backend - 404
```
## 给kubernetes部署tls数字证书
数字证书是自签名的泛域名证书，生成*.wayne.com的CN，需要用到server.key 和server.csr私钥和自签名证书
```
 mkdir -p CA/{certs,crl,newcerts,private}
touch CA/index.txt
echo 00 > CA/serial
 cp /etc/pki/tls/openssl.cnf .
openssl req -new -x509 -days 3650 -keyout ca.key -out ca.crt -config openssl.cnf
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config openssl.cnf
 openssl ca -days 180 -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -config openssl.cnf

 kubectl create secret tls frognew-com-tls-secret --cert=server.crt --key=server.key -n kube-system
```
后边部署在kube-system命名空间中的dashboard要使用这个证书，因此这里同样在kube-system中创建证书的secret
## 使用Helm部署dashboard
kubernetes-dashboard.yaml：
```
ingress:
  enabled: true
  hosts: 
    - k8s.wayne.com
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/secure-backends: "true"
  tls:
    - secretName: frognew-com-tls-secret
      hosts:
      - k8s.wayne.com
rbac:
  clusterAdminRole: true
helm install stable/kubernetes-dashboard \
-n kubernetes-dashboard \
--namespace kube-system  \
-f kubernetes-dashboard.yaml
kubectl -n kube-system get secret | grep kubernetes-dashboard-token
kubernetes-dashboard-token-tjj25                 kubernetes.io/service-account-token   3         37s
```
```
kubectl describe -n kube-system secret/kubernetes-dashboard-token-tjj25
Name:         kubernetes-dashboard-token-tjj25
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard
              kubernetes.io/service-account.uid=d19029f0-9cac-11e8-8d94-080027db403a

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi10amoyNSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImQxOTAyOWYwLTljYWMtMTFlOC04ZDk0LTA4MDAyN2RiNDAzYSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.w1HZrtBOhANdqSRLNs22z8dQWd5IOCpEl9VyWQ6DUwhHfgpAlgdhEjTqH8TT0f4ftu_eSPnnUXWbsqTNDobnlxet6zVvZv1K-YmIO-o87yn2PGIrcRYWkb-ADWD6xUWzb0xOxu2834BFVC6T5p5_cKlyo5dwerdXGEMoz9OW0kYvRpKnx7E61lQmmacEeizq7hlIk9edP-ot5tCuIO_gxpf3ZaEHnspulceIRO_ltjxb8SvqnMglLfq6Bt54RpkUOFD1EKkgWuhlXJ8c9wJt_biHdglJWpu57tvOasXtNWaIzTfBaTiJ3AJdMB_n0bQt5CKAUnKBhK09NP3R0Qtqog
```
在dashboard的登录窗口使用上面的token登录。
![dashboard](/images/kubernetes-dashboard.png)
# 参考
[centos7下终端使用代理](https://blog.csdn.net/u012375924/article/details/78706910)  
[docker 使用 socks5代理](https://www.jianshu.com/p/fef11e46ebf1)  
[使用kubeadm安装kubernetes1.11](https://blog.frognew.com/2018/08/kubeadm-install-kubernetes-1.11.html)  
