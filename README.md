# kubernetes-
## 介绍 kubernetes 的几种安装方式（新手上路，大家有其他的方式欢迎指导）
* kubeadm
* 源码

### kubeadm 与 源码方式
kubeadm 方式可以可以在线安装和离线安装，这里所谓的离线和在线指的是kubernetes 组件已经下载至本地或者直接通过网络安装（可能需要翻墙）。建议使用离线方式安装。

源码安装即是下载kubernetes官方编译好的二进制源码进行安装。

### kubernetes 组件介绍


#### kubernetes 安装前准备
* 这里使用6个节点的安装方式，即3个master和3个node组成的集群。
