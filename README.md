# kubernetes-
## 介绍 kubernetes 的几种安装方式（新手上路，大家有其他的方式欢迎指导）
* kubeadm
* 源码

### kubeadm 与 源码方式
kubeadm 方式可以可以在线安装和离线安装，这里所谓的离线和在线指的是kubernetes 组件已经下载至本地或者直接通过网络安装（可能需要翻墙）。建议使用离线方式安装。

源码安装即是下载kubernetes官方编译好的二进制源码进行安装。

### kubernetes 组件介绍
* kubernetes 主要由以下几个核心组件组成：
    * etcd 保存了整个集群的状态，非常重要，如果数据丢失那么集群将无法恢复；因此高可用部署集群首先就是etcd是高可用。时常进行数据备份。
    * apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
    * controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
    * scheduler 负责资源的调度，按照预定的调度策略将pod调度到相应的机器上；
    * kubelet 负责维护容器的生命周期，同时也负责Volume（cvi）和网络（CNI）的管理；负责与node上的docker engine打交道；
    * container runtime 负责镜像管理以及pod和容器的真正运行（CRI）；
    * kube-proxy 负责为Service提供cluster内部的服务发现和负载均衡，当前主要通过设置iptables规则实现。

#### kubernetes 安装前准备
* 这里使用6个节点的安装方式，即3个master和3个node组成的集群。

**集群配置清单**

所有服务器安装完成后进行初始化
```
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum install -y lrzsz unzip nc ntp telnet openssh-clients git vim libselinux-python net-tools wget ftp net-tools wget
```

host|role|service|configure|ip
|:-:|:-:|:-:|:-:|-:
k8s-master-01| master | apiserver、etcd、contoller manage、scheduler、kubelet、keepalived | 2 core 4GB | 自行规划
k8s-master-02| master | apiserver、etcd、contoller manage、scheduler、kubelet、keepalived | 1 core 1GB | 自行规划
k8s-master-03| master | apiserver、etcd、contoller manage、scheduler、kubelet、keepalived | 1 core 1GB | 自行规划
k8s-node-01| node | kubelet、kube-proxy | 1 core 1GB | 自行规划
k8s-node-02| node | kubelet、kube-proxy | 1 core 1GB | 自行规划
k8s-node-03| node | kubelet、kube-proxy | 1 core 1GB | 自行规划

* 所有虚拟机在同一个内网中
* 配置一个docker hub仓库，这是使用horbar实现
* 一个Apache web服务器，用作离线文件的存储，便于脚本的调用。如 http://example.com，这里未设置HTTPS。如果需要，请自行配置，并在下面将HTTP换成https。

### kubernetes master 部署
**一、 在所有的master服务器上部署docker并对服务器做相应配置**
1. 关闭防火墙和selinux
    ```
    systemctl stop firewalld
    systemctl disable firewalld
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    setenforce 0
    ```
2. 安装基础软件
    ```
    yum install -y keepalived chrony ntp
    systemctl enable chronyd
    systemctl restart chronyd
    ```

3. docker 安装

    docker engine下载地址（也可以设置为docker下载YUM源）：
    ```
    https://yum.dockerproject.org/repo/main/centos/7/Packages/
    ```

    或
    ```
    tee /etc/yum.repos.d/docker.repo << 'EOF'
    [dockerrepo]
    name=Docker Repository
    baseurl=https://yum.dockerproject.org/repo/main/centos/7/
    enabled=1
    gpgcheck=1
    gpgkey=https://yum.dockerproject.org/gpg
    EOF
    ```
    docker 正式安装：
    ```
    yum install https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-1.12.6-1.el7.centos.x86_64.rpm
    yum install -y https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-selinux-1.12.6-1.el7.centos.noarch.rpm
    systemctl enable docker.service
    systemctl restart docker.service
    ```
    **小提示：**
    ```
    由于下载外网镜像时速度过慢的问题，可以使用阿里云的镜像加速器服务。
    使用 systemctl enable docker (在使用docker-engine时生效) 启动后，在配置文件/etc/systemd/system/multi-user.target.wants/docker.service ExecStart=行后添加镜像加速地址（该地址可以在注册或者是使用阿里云账号登陆之后获得docker镜像加速地址）如：

    ExecStart=/usr/bin/dockerd --registry-mirror=https://rkueuygt.mirror.aliyuncs.com

    systemctl daemon-reload
    systemctl restart docker
    ```
4. 配置host
    ```
    echo "192.168.174.132 k8s-master-01" >>/etc/hosts
    echo "192.168.174.133 k8s-master-02" >>/etc/hosts
    echo "192.168.174.134 k8s-master-03" >>/etc/hosts
    ```

5. 配置kubernetes 所有节点iptables
    ```
    cat <<EOF > /etc/sysctl.d/kubernetes.conf
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl -p /etc/sysctl.d/kubernetes.conf
    ```

5. 安装etcd和cfssl
    ```
    cd /usr/local/bin/
    wget http://192.168.174.130/docker/k8s/etcd-3.2.11/etcd
    wget http://192.168.174.130/docker/k8s/etcd-3.2.11/etcdctl
    chmod a+x etcd*

    wget http://192.168.174.130/docker/ssl/cfssl
    wgethttp://192.168.174.130/docker/ssl/cfssljson
    wget http://192.168.174.130/docker/ssl/cfssl-certinfo
    chmod a+x  cfssl*
    ```

6. 配置keepalive
    ```
    cat <<EOF > /etc/keepalived/check_apiserver.sh 
    #!/bin/bash
    err=0
    for k in $( seq 1 10 )
    do
        check_code=$(ps -ef|grep kube-apiserver | wc -l)
        if [ "$check_code" = "1" ]; then
            err=$(expr $err + 1)
            sleep 5
            continue
        else
            err=0
            break
        fi
    done
    if [ "$err" != "0" ]; then
        echo "systemctl stop keepalived"
        /usr/bin/systemctl stop keepalived
        exit 1
    else
        exit 0
    fi
    EOF
    chmod +x check_apiserver.sh
    cd /etc/keepalived/
    >keepalived.conf
    systemctl enable keepalived && systemctl restart keepalived
    ```




