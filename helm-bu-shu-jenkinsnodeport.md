# Helm部署Jenkins--NodePort方式
## Helm下载Jenkins Chart文件
```
helm search jenkins
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                                           stable/jenkins	0.18.1       	2.121.3    	Open source continuous integration server.
helm fetch stable/jenkins
ls
jenkins-0.18.1.tgz
```
## 修改Chart文件，设置成NodePort方式
* Master.ServiceType默认是LoadBalancer，如果是NodePort方式需要修改成NodePort，如果是Ingress需要修改成ClusterIP
* Persistence.StorageClass 根据集群的StorageClass设置。集群中响应namespace需要有StorageClass的secret
* rbac.install默认是false，要用到ServiceAccount调度k8s集群，需要设置成true
## 部署Jenkins，设置Jenkins运行
修改Jenkins URL  默认的是ClusterIP名称+port方式，在Jenkins的系统设置页面会报`反向代理有误`。把URL设置成NodePort的地址。

## Jenkins常见问题及故障排查
## Jenkins调度Slave执行任务，需要等待一分钟。  
在Master节点处显示等待日志如下：
```
Excess workload after pending Kubernetes agents: 1

Oct 13, 2018 5:22:40 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesCloud provision

Template: Kubernetes Pod Template

Oct 13, 2018 5:22:40 AM INFO okhttp3.internal.platform.Platform log

ALPN callback dropped: HTTP/2 is disabled. Is alpn-boot on the boot class path?

Oct 13, 2018 5:22:40 AM INFO hudson.slaves.NodeProvisioner$StandardStrategyImpl apply

Started provisioning Kubernetes Pod Template from kubernetes with 1 executors. Remaining excess workload: 0

Oct 13, 2018 5:22:50 AM INFO hudson.slaves.NodeProvisioner$2 run

Kubernetes Pod Template provisioning successfully completed. We have now 2 computer(s)

Oct 13, 2018 5:22:50 AM INFO okhttp3.internal.platform.Platform log

ALPN callback dropped: HTTP/2 is disabled. Is alpn-boot on the boot class path?

Oct 13, 2018 5:22:50 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Created Pod: default-mbgzn in namespace default

Oct 13, 2018 5:22:50 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for Pod to be scheduled (0/100): default-mbgzn

Oct 13, 2018 5:22:56 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (0/100): default-mbgzn

Oct 13, 2018 5:22:57 AM INFO hudson.TcpSlaveAgentListener$ConnectionHandler run

Accepted JNLP4-connect connection #3 from 10.1.105.238/10.1.105.238:59250

Oct 13, 2018 5:22:57 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (1/100): default-mbgzn

Oct 13, 2018 5:22:58 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (2/100): default-mbgzn

Oct 13, 2018 5:22:59 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (3/100): default-mbgzn

Oct 13, 2018 5:23:00 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (4/100): default-mbgzn

Oct 13, 2018 5:23:01 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (5/100): default-mbgzn

Oct 13, 2018 5:23:02 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (6/100): default-mbgzn

Oct 13, 2018 5:23:03 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (7/100): default-mbgzn

Oct 13, 2018 5:23:04 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (8/100): default-mbgzn

Oct 13, 2018 5:23:05 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (9/100): default-mbgzn

Oct 13, 2018 5:23:06 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (10/100): default-mbgzn

Oct 13, 2018 5:23:08 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (11/100): default-mbgzn

Oct 13, 2018 5:23:09 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (12/100): default-mbgzn

Oct 13, 2018 5:23:10 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (13/100): default-mbgzn

Oct 13, 2018 5:23:11 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (14/100): default-mbgzn

Oct 13, 2018 5:23:12 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (15/100): default-mbgzn

Oct 13, 2018 5:23:13 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (16/100): default-mbgzn

Oct 13, 2018 5:23:14 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (17/100): default-mbgzn

Oct 13, 2018 5:23:15 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (18/100): default-mbgzn

Oct 13, 2018 5:23:16 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (19/100): default-mbgzn

Oct 13, 2018 5:23:17 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (20/100): default-mbgzn

Oct 13, 2018 5:23:18 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (21/100): default-mbgzn

Oct 13, 2018 5:23:19 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (22/100): default-mbgzn

Oct 13, 2018 5:23:20 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (23/100): default-mbgzn

Oct 13, 2018 5:23:21 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (24/100): default-mbgzn

Oct 13, 2018 5:23:22 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch

Waiting for agent to connect (25/100): default-mbgzn

Oct 13, 2018 5:23:56 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave _terminate

Terminating Kubernetes instance for agent default-mbgzn

Oct 13, 2018 5:23:57 AM INFO okhttp3.internal.platform.Platform log

ALPN callback dropped: HTTP/2 is disabled. Is alpn-boot on the boot class path?

Oct 13, 2018 5:23:57 AM INFO org.jenkinsci.plugins.workflow.job.WorkflowRun finish

pipeline-1 #3 completed: SUCCESS

Oct 13, 2018 5:23:57 AM WARNING hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor monitorDetailed

Failed to monitor default-mbgzn for Free Temp Space
java.util.concurrent.TimeoutException
	at hudson.remoting.Request$1.get(Request.java:316)
	at hudson.remoting.Request$1.get(Request.java:240)
	at hudson.remoting.FutureAdapter.get(FutureAdapter.java:59)
	at hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor.monitorDetailed(AbstractAsyncNodeMonitorDescriptor.java:114)
	at hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor.monitor(AbstractAsyncNodeMonitorDescriptor.java:76)
	at hudson.node_monitors.AbstractNodeMonitorDescriptor$Record.run(AbstractNodeMonitorDescriptor.java:305)

Oct 13, 2018 5:23:57 AM WARNING hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor monitorDetailed

Failed to monitor default-mbgzn for Free Disk Space
java.util.concurrent.TimeoutException
	at hudson.remoting.Request$1.get(Request.java:316)
	at hudson.remoting.Request$1.get(Request.java:240)
	at hudson.remoting.FutureAdapter.get(FutureAdapter.java:59)
	at hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor.monitorDetailed(AbstractAsyncNodeMonitorDescriptor.java:114)
	at hudson.node_monitors.AbstractAsyncNodeMonitorDescriptor.monitor(AbstractAsyncNodeMonitorDescriptor.java:76)
	at hudson.node_monitors.AbstractNodeMonitorDescriptor$Record.run(AbstractNodeMonitorDescriptor.java:305)

Oct 13, 2018 5:24:02 AM WARNING jenkins.slaves.DefaultJnlpSlaveReceiver channelClosed

Computer.threadPoolForRemoting [#49] for default-mbgzn terminated
java.nio.channels.ClosedChannelException
	at org.jenkinsci.remoting.protocol.impl.ChannelApplicationLayer.onReadClosed(ChannelApplicationLayer.java:209)
	at org.jenkinsci.remoting.protocol.ApplicationLayer.onRecvClosed(ApplicationLayer.java:222)
	at org.jenkinsci.remoting.protocol.ProtocolStack$Ptr.onRecvClosed(ProtocolStack.java:832)
	at org.jenkinsci.remoting.protocol.FilterLayer.onRecvClosed(FilterLayer.java:287)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.onRecvClosed(SSLEngineFilterLayer.java:181)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.switchToNoSecure(SSLEngineFilterLayer.java:283)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.processWrite(SSLEngineFilterLayer.java:503)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.processQueuedWrites(SSLEngineFilterLayer.java:248)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.doSend(SSLEngineFilterLayer.java:200)
	at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.doCloseSend(SSLEngineFilterLayer.java:213)
	at org.jenkinsci.remoting.protocol.ProtocolStack$Ptr.doCloseSend(ProtocolStack.java:800)
	at org.jenkinsci.remoting.protocol.ApplicationLayer.doCloseWrite(ApplicationLayer.java:173)
	at org.jenkinsci.remoting.protocol.impl.ChannelApplicationLayer$ByteBufferCommandTransport.closeWrite(ChannelApplicationLayer.java:314)
	at hudson.remoting.Channel.close(Channel.java:1450)
	at hudson.remoting.Channel.close(Channel.java:1403)
	at hudson.slaves.SlaveComputer.closeChannel(SlaveComputer.java:821)
	at hudson.slaves.SlaveComputer.access$800(SlaveComputer.java:105)
	at hudson.slaves.SlaveComputer$3.run(SlaveComputer.java:737)
	at jenkins.util.ContextResettingExecutorService$1.run(ContextResettingExecutorService.java:28)
	at jenkins.security.ImpersonatingExecutorService$1.run(ImpersonatingExecutorService.java:59)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

Oct 13, 2018 5:24:02 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave deleteSlavePod

Terminated Kubernetes instance for agent default/default-mbgzn

Oct 13, 2018 5:24:02 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave _terminate

Disconnected computer default-mbgzn
```
查看Pod的工作日志，发现Slave节点的工作细节：
```
[root@icp-102 jenkins]# kubectl  logs default-mbgzn 
Warning: JnlpProtocol3 is disabled by default, use JNLP_PROTOCOL_OPTS to alter the behavior
Warning: SECRET is defined twice in command-line arguments and the environment variable
Warning: AGENT_NAME is defined twice in command-line arguments and the environment variable
Oct 13, 2018 5:22:54 AM hudson.remoting.jnlp.Main createEngine
INFO: Setting up slave: default-mbgzn
Oct 13, 2018 5:22:55 AM hudson.remoting.jnlp.Main$CuiListener <init>
INFO: Jenkins agent is running in headless mode.
Oct 13, 2018 5:22:55 AM hudson.remoting.Engine startEngine
WARNING: No Working Directory. Using the legacy JAR Cache location: /home/jenkins/.jenkins/cache/jars
Oct 13, 2018 5:22:56 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Locating server among [http://bunking-elk-jenkins:8080]
Oct 13, 2018 5:22:56 AM org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver resolve
INFO: Remoting server accepts the following protocols: [JNLP4-connect, Ping]
Oct 13, 2018 5:22:56 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Agent discovery successful
  Agent address: bunking-elk-jenkins-agent
  Agent port:    50000
  Identity:      ed:d7:5c:18:4c:e6:b1:cc:49:27:97:bd:b2:fe:a0:e7
Oct 13, 2018 5:22:57 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Handshaking
Oct 13, 2018 5:22:57 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Connecting to bunking-elk-jenkins-agent:50000
Oct 13, 2018 5:22:57 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Trying protocol: JNLP4-connect
Oct 13, 2018 5:22:58 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Remote identity confirmed: ed:d7:5c:18:4c:e6:b1:cc:49:27:97:bd:b2:fe:a0:e7
Oct 13, 2018 5:23:03 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Connected
Oct 13, 2018 5:24:02 AM org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave$SlaveDisconnector call
INFO: Disabled slave engine reconnects.
```
问题原因分析：

## Master节点报OOMKilled问题
