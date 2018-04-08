# kubernetes-
## 介绍 kubernetes 的几种安装方式（新手上路，大家有其他的方式欢迎指导）
* kubeadm
* 源码

### kubeadm 与 源码方式
kubeadm 方式可以可以在线安装和离线安装，这里所谓的离线和在线指的是kubernetes 组件已经下载至本地或者直接通过网络安装（可能需要翻墙）。建议使用离线方式安装。

源码安装即是下载kubernetes官方编译好的二进制源码进行安装。

### kubernetes 组件介绍
* kubernetes 主要由以下几个核心组件组成
    * etcd 保存了整个集群的状态；
    * apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
    * controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
    * scheduler 负责资源的调度，按照预定的调度策略将pod调度到相应的机器上；
    * kubelet 负责维护容器的生命周期，同时也负责Volume（cvi）和网络（CNI）的管理；
    * container runtime 负责镜像管理以及pod和容器的真正运行（CRI）；
    * kube-proxy 负责为Service提供cluster内部的服务发现和负载均衡。

#### kubernetes 安装前准备
* 这里使用6个节点的安装方式，即3个master和3个node组成的集群。

**集群配置清单**

host|role|service|configure
-|:-:|-:|-:|-:|-:


