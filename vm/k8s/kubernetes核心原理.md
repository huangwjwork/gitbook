# kubernetes核心原理

## kubernetes API

提供各种资源对象（Pod、RC、Service、deployment、HPA、PV等等的）增删改查及watch等REST接口，成为集群内各个功能模块之间的数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心。

* 集群管理的API入口
* 是资源配额控制的入口
* 提供了完备的集群安全机制

### 集群功能模块之间的通信

集群所有模块都只和kubeAPI通信，不直接操作etcd数据库

如kubelet：kubelet每个一个周期就与api通信报告自身状态，api收到信息后，将节点状态存入etcd中；kubelet通过watch api监听pod信息，如果坚挺到新的pod副本被调度到本节点，则执行pod相应的创建和启动逻辑。删除亦如此

一个完整的pod创建过程

1. kubectl创建一个pod，将pod信息传递给api
1. api将pod信息写入etcd
1. kube-controller-maneger通过api watch接口实时监控pod变化信息，做相关操作
1. kube-schduler与api交互，监控到pod新建信息，检索符合pod要求的node列表，开始执行pod调度逻辑，将pod绑定到目标节点
1. kubelet与API交互，检测到pod创建信息，开始创建pod

## Controller Manager

负责集群内node、pod、endpoint、namespace、serviceaccount、资源配额等管理，当某个node宕机时，controller manager会及时发现故障并执行自动化修复流程

* ReplicationController
  1. 控制pod副本数量符合预期
  1. 通过调整spec.replicas来实现系统扩容和缩容
  1. 实现滚动升级
* NodeController
  1. 设置CIDR，防止节点间CIDR冲突
  1. 逐个读取节点信息，实时修改nodeStatusMap，状态没变化则修改nodestatusmap的时间，否则修改nodestatusmap的信息
* ResourceQuotaController
* NamespaceController
* ServiceAccountController
* TokenController
* ServiceController
* EndpointController

controller通过api实施监控整个集群里的每个资源对象的当前状态，当发生故障时会尝试将系统修复至期望状态

## Scheduler

将待调度的pod按照特定的调度算法和调度策略绑定到集群中的某个特定的node，并将绑定信息写入etcd。

默认调度流程分如下两步：

1. 预选调度过程，便利所有目标node，筛选出符合要求的候选节点
1. 确定最优节点，在第一步的基础上，采用优选策略计算出每一个候选节点的积分，积分最高者胜出

### 预选调度策略

#### NoDiskConflict

读取备选pod的所有volume信息（**Pod.Spec.Volumes**），如果volume是GCE或者AWS，且所要调度的节点上存在已挂载相同volume的pod，则存在磁盘冲突，不适合备选pod

#### PodFitsResources

检测备选pod和备选node是否存在资源需求冲突

#### PodSelectorMatches

判断备选node是否包含备选pod的标签指定的标签（**spec.nodeSelector**）

#### PodFitsHost

判断Pod的**spec.nodeName**所指定的节点名称是否和备选节点的名称是否一致

#### CheckNodeLablePresence

scheduler会通过**RegisterCustomfitPredicate**注册该策略，判定判断策略列出的标签存在时是否选择该备选节点

如果presence为false，存在标签时为false

如果presence为true，存在标签使为true

#### CheckServiceAffinity



#### PodFitsPorts

判断备选pod所用的端口列表是否在备选节点被占用

### 优选调度策略

#### LeastRequestedPriority

该优选策略用于从备选node列表中选出资源消耗最小的node

1. 计算备选node上的totalMillionCPU
1. 计算备选node上的totalMemory
1. 计算得分

#### CalculateNodeLabelPriority

label优先级，

presence值为true，label匹配，score为10

presence值为false，label不匹配，score为10

####  BalanceResourceAllocation

从备选节点中选则各项资源使用率最均衡的节点，计算方式与LeastRequestedPriority略微不同

## kubelet

### 节点管理

向apiserver注册自己，并定时向apiserver发送node的新消息，默认10秒一次，可以通过`node-status-update-frequency`设置时间间隔

### Pod管理

1. 定期检查配置文件/etc/kubernetes/manifests/

1. apiserver：kubelet通过apiserver监听etcd，同步pod列表，对pod做相应处理

   `关于staticpod，不受apiserver监管，apiserver会创建一个mirror pod与其相匹配，mirrorpod会真实反映staticpod的状态，当staticpod被删除，mirrorpod也会被删除`

#### pod创建过程

kubelet监听到创建信息后，做如下操作

1. 为pod创建一个数据目录
1. 从apiserver读取pod清单
1. 为pod挂载外部卷
1. 下载pod用到的secret
1. 检查pod容器是否正常启动，检查pause容器，如果pause容器没有启动，则停止pod里所有的容器进程，如果在pod中有需要删除的容器，则点出这些容器
1. 用kubernetes/pause镜像为每个pod创建pause容器，用于接管pod中其他容器的网络
1. 
   1. 为容器计算hash值，然后根据容器名查询对应docker容器的hash，若不匹配，则停止docker容器中的进程，并停止pause容器
   1. 若容器终止，且没有restartPolicy，则不做任何处理
   1. 调用dockerclient下载镜像，调用dockerclient运行容器

### 容器健康检查

### cAdvisor资源监控

cAdvisor是一个开源的分析容器资源使用率和性能特性的代理工具，自动查找所在节点上的容器，采集CPU，内存，文件系统和网络使用的统计信息

通过所在节点的4194端口暴露一个简单的UI，暴露node信息，docker信息，pod信息等等

## kube-proxy

service实现的基础，将对service的访问转发到后端的多个pod实例上

转发实现原理，IPtables，kube-proxy动态创建于service相关的iptables规则，实现了将clusterIP即nodeport的请求流量重定向到kube-proxy进程是对应服务的代理端口的功能

无论是clusterIP+targetport还是nodeIP+nodeport，都重定向到kube-proxy坚挺的service服务代理端口

kube-proxy默认roundrobin轮询，但也可以做session保持。若session没有超时，则保存在affinitystate中的IP后来的访问都会转发到同一个pod

kube-proxy通过查询和监听apiserver中service与endpoint的变化，为每个service都建立了一个服务代理对象，并自动同步。包含了一个用于监听此服务请求的socketserver，socketserver的端口是随机选择的一个本地空闲端口。同时，kube-proxy创建一个loadbalance，保存了service到对应的后端endpoint列表的动态转发路由表。



## 集群安全机制

