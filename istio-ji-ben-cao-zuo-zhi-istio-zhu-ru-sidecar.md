# Istio基本操作之Istio注入Sidecar
# 注入
## 手工注入Sidecar
手工注入过程会修改控制器（例如deployment）的配置。通过修改pod template的手段，把Sidecar注入到目标控制器生成的所有pod之中。  
使用集群内置配置将Sidecar注入到Deployment中：
```
$ istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -

[root@node1 istio-1.0.2]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
sleep-7dc8675845-rw548   2/2     Running   0          14m
[root@node1 istio-1.0.2]# kubectl get deployment sleep -o wide
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS          IMAGES                                     SELECTOR
sleep   1         1         1            1           14m   sleep,istio-proxy   tutum/curl,docker.io/istio/proxyv2:1.0.2   app=sleep

[root@node1 istio-1.0.2]# kubectl delete -f samples/sleep/sleep.yaml 
service "sleep" deleted
deployment.extensions "sleep" deleted
```
查看生成的sleep pod，发现sleep pod中包含两个容器。对当前生成的deployment进行删除。  
## 应用部署
部署sleep应用，检查一下是不是只是生成一个容器。
```
[root@node1 istio-1.0.2]# kubectl apply -f samples/sleep/sleep.yaml 
service/sleep created
deployment.extensions/sleep created
[root@node1 istio-1.0.2]# kubectl get deployment -o wide
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                    SELECTOR
sleep   1         1         1            1           16s    sleep        tutum/curl                app=sleep

[root@node1 istio-1.0.2]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
sleep-7549f66447-rw5sp   1/1     Running   0          63s
```
给当前命名空间设置标签：istio-injection=enabled:
```
[root@node1 istio-1.0.2]# kubectl get namespace -L istio-injection
NAME            STATUS   AGE    ISTIO-INJECTION
default         Active   3d8h   
ingress-nginx   Active   3d4h   
istio-system    Active   2d     
kube-public     Active   3d8h   
kube-system     Active   3d8h   
[root@node1 istio-1.0.2]# kubectl label namespace default istio-injection=enabled
namespace/default labeled
[root@node1 istio-1.0.2]# kubectl get namespace -L istio-injection
NAME            STATUS   AGE    ISTIO-INJECTION
default         Active   3d8h   enabled
ingress-nginx   Active   3d4h   
istio-system    Active   2d     
kube-public     Active   3d8h   
kube-system     Active   3d8h
```
自动注入设置之后，在创建pod时触发Sidecar的注入过程。按照教程应该会生成两个pod。但是实际情况是pod删除之后就不能生成新的pod了。
```
[root@node1 istio-1.0.2]# kubectl get deployment
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sleep   1         0         0            0           7m31s
[root@node1 istio-1.0.2]# kubectl get po
No resources found.

[root@node1 istio-1.0.2]# kubectl describe deployment sleep
Name:                   sleep
Namespace:              default
CreationTimestamp:      Thu, 04 Oct 2018 23:54:42 +0800
Labels:                 app=sleep
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"sleep","namespace":"default"},"spec":{"replica...
Selector:               app=sleep
Replicas:               1 desired | 0 updated | 0 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  app=sleep
  Containers:
   sleep:
    Image:      tutum/curl
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sleep
      infinity
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type             Status  Reason
  ----             ------  ------
  Available        True    MinimumReplicasAvailable
  ReplicaFailure   True    FailedCreate
OldReplicaSets:    <none>
NewReplicaSet:     sleep-7549f66447 (0/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  8m    deployment-controller  Scaled up replica set sleep-7549f66447 to 1
```
查看sleep副本集Events
发现报错
```
Error creating: Internal error occurred: failed calling admission webhook "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject?timeout=30s: EOF
```
问题描述issue
```
https://github.com/istio/istio/issues/7233
```

然后删除当前namespace的自动注入，就可以创建pod

