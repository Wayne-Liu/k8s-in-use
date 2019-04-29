## This CKA exam curriculum includes these general domains and their weights on the exam:
● Application Lifecycle Management 8%  
● Installation, Configuration & Validation 12%  
● Core Concepts 19%  
● Networking 11%  
● Scheduling 5%  
● Security 12%  
● Cluster Maintenance 11%  
● Logging / Monitoring 5%  
● Storage 7%  
● Troubleshooting 10%  

```
在3~4小时内用命令行进行排障， 解决问题， 相关知识点和权重

Installation, Configuration & Validation 安装，配置和验证12%
设计一个k8s 集群
安装k8s master 和 nodes
配置安全的集群通信
配置高可用的k8s集群
知道如何获取k8s的发行的二进制文件
提供底层的基础措施来部署一个集群
选择一个网络方案
选择你的基础设施配置
在你的集群上配置端对端的测试
分析端对端测试结果
运行节点的端对端测试
Core Concepts 核心概念 19%
理解k8s api原语
理解k8s 架构
理解services和其它网络相关原语
Application Lifecycle Management 应用生命周期管理 8%
理解Deployment， 并知道如何进行rolling update 和 rollback
知道各种配置应用的方式
知道如何为应用扩容
理解基本的应用自愈相关的内容
Networking 网络 11%
理解在集群节点上配置网络
理解pod的网络概念
理解service networking
部署和配置网络负载均衡器
知道如何使用ingress 规则
知道如何使用和配置cluster dns
理解CNI
Storage 存储 7%
理解持久化卷（pv），并知道如何创建它们
理解卷（volumes）的access mode
理解持久化卷声明（pvc）的原语
理解k8s的存储对象（kubernetes storage objects）
知道如何为应用配置持久化存储
Scheduling 调度 5%
使用label选择器来调度pods
理解Daemonset的角色
理解resource limit 会如何影响pod 调度
理解如何运行多个调度器， 以及如何配置pod使用它们
不使用调度器， 手动调度一个pod
查看和显示调度事件events
知道如何配置kubernetes scheduler
Security 安全 12%
知道如何配置认证和授权
理解k8s安全相关原语
理解如何配置网络策略（network policies）
配合使用镜像的安全性
定义安全上下文
安全的持久化保存键值
Cluster Maintenance 集群维护 11%
理解k8s的集群升级过程
促进操作系统的升级
补充备份和还原的方法论
Logging / Monitoring 日志/监控 5%
理解如何监控所有的集群组件
理解如何监控应用
管理集群组件日志
管理应用日志
Troubleshooting 问题排查 10%
排查应用失败故障
排查控制层（control panel）故障
排查工作节点（work node）故障
排查网络故障
```