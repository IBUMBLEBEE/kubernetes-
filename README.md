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
* kubernetes 非核心组件
    * CNI  容器网络接口，负责将pod挂载到flannel网络上的。
    * kube-dns kubernetes域名解析服务
    * dashboard kubernetes UI页面
    * heapster 监控组件（结合grafana + influxdb + heapster 另外方案：prometheus）

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

**温馨提示**
* 所有虚拟机在同一个内网中
* 配置一个docker hub仓库，这是使用horbar实现（在内网中安装kubernetes时，这一步是很重要的一步，请重视）。horbar 在官网有离线安装包，具体的安装步骤请按官方步骤进行。
* 一个Apache web服务器，用作离线文件的存储，便于脚本的调用。如 http://example.com，这里未设置HTTPS。如果需要，请自行配置，并在下面将HTTP换成https。

### kubernetes master 部署
#### 一、 在所有的master服务器上部署docker并对服务器做相应配置
1. 关闭防火墙和selinux
    ```
    systemctl stop firewalld
    systemctl disable firewalld
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
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

    yum makecache fast
    ```
    docker 正式安装：
    ```
    yum install -y docker-1.12.6
    systemctl start docker.service
    systemctl enable docker.service
    ```
    **小提示（设置阿里云镜像加速器）**
    ```
    由于下载外网镜像时速度过慢的问题，可以使用阿里云的镜像加速器服务。
    使用 systemctl enable docker (在使用docker-engine时生效) 启动后，在配置文件/etc/systemd/system/multi-user.target.wants/docker.service ExecStart=行后添加镜像加速地址（该地址可以在注册或者是使用阿里云账号登陆之后获得docker镜像加速地址）如：

    ExecStart=/usr/bin/dockerd --registry-mirror=https://rkueuygt.mirror.aliyuncs.com

    systemctl daemon-reload
    systemctl restart docker
    ```
4. 配置host
    ```
    echo "192.168.223.160 k8s-master-001" >>/etc/hosts
    echo "192.168.223.161 k8s-master-002" >>/etc/hosts
    echo "192.168.223.162 k8s-master-003" >>/etc/hosts
    ```

5. 配置kubernetes 所有节点iptables
    ```
    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system
    ```

5. 安装etcd和cfssl
    这是内网的web服务器，etcd二进制文件可在GitHub下载，etcd版本可根据kubernetes官网推荐的etcd版本进行选择
    ```
    cd /usr/local/bin/
    wget http://192.168.233.134/docker/etcd/etcd
    wget http://192.168.233.134/docker/etcd/etcdctl
    chmod a+x etcd*

    curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    chmod +x /usr/local/bin/cfssl*
    ```

6. 配置keepalive(这是自定义的，检测apiserver服务的keepalived脚本。kubernetes官网有配置好的，请阅读https://kubernetes.io/docs/setup/independent/high-availability/配置并自行配置)
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
    chmod +x /etc/keepalived/check_apiserver.sh
    >/etc/keepalived/keepalived.conf
    systemctl enable keepalived && systemctl restart keepalived
    ```

#### 二、创建验证
1. 创建CA证书配置，生成CA证书和私钥（该服务器证书一年有效期）
    先用cfssl命令生成包含默认配置的config.json和csr.json文件
    ```
    mkdir /opt/ssl
    cd /opt/ssl
    ```
    config.json 文件
    ```
    cat << EOF> config.json
    {
      "signing": {
            "default": {
                  "expiry":  "87600h"    
            },
            "profiles": {
                  "kubernetes": {
                        "usages": [
                                "signing",
                                "key encipherment",
                                "server auth",
                                "client auth"        
                        ],
                        "expiry":  "87600h"      
                    }    
            }  
        }
    }
    EOF
    ```
    创建car.json文件
    ```
    cat <<EOF > csr.json
    {
      "CN":  "kubernetes",
      "key": {
            "algo":  "rsa",
            "size":  2048  
        },
      "names": [
            {
                  "C":  "CN",
                  "ST":  "ShenZhen",
                  "L":  "ShenZhen",
                  "O":  "k8s",
                  "OU":  "System"    
            }  
        ]
    }
    EOF
    ```

2. 创建ETCD证书配置（etcd秘钥建议不要放在/etc/kubernetes/pki目录下，如果你要精力重新生成etcd秘钥信息）
    在/opt/ssl下添加文件etcd-csr.json（配置对应的etcd服务器地址），内容如下：
    ```
    cat <<EOF> etcd-csr.json
    {
      "CN":  "etcd",
      "hosts": [
            "127.0.0.1",
            "192.168.233.128",
            "192.168.233.129",
            "192.168.233.130"  
    ],
      "key": {
            "algo":  "rsa",
            "size":  2048  
    },
      "names": [
            {
                  "C":  "CN",
                  "ST":  "ShenZhen",
                  "L":  "ShenZhen",
                  "O":  "k8s",
                  "OU":  "System"    
        }  
    ]
    }
    EOF
    ```
    3. 创建kube-apiserver证书（添加kubernetes master apiserver节点以及keepalived虚拟IP）
        ```
        cat <<EOF> kubernetes-csr.json
        {
        "CN": "kubernetes",
        "hosts": [
            "127.0.0.1",
            "192.168.233.128",
            "192.168.233.129",
            "192.168.233.130",
            "192.168.233.134",
            "10.221.0.1",
            "kubernetes",
            "kubernetes.default",
            "kubernetes.default.svc",
            "kubernetes.default.svc.cluster",
            "kubernetes.default.svc.cluster.local"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "ST": "ShenZhen",
            "L": "ShenZhen",
            "O": "k8s",
            "OU": "System"
            }
        ]
        }
        EOF
        ```
    4. 创建kube-admin证书
        ```
        cat <<EOF>admin-csr.json
        {
        "CN": "admin",
        "hosts": [],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "ST": "ShenZhen",
            "L": "ShenZhen",
            "O": "system:masters",
            "OU": "System"
            }
        ]
        }
        EOF
        ```
    5. 创建kube-proxy证书
        ```
        cat <<EOF>kube-proxy-csr.json
        {
            "CN": "system:kube-proxy",
            "hosts": [],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                "C": "CN",
                "ST": "ShenZhen",
                "L": "ShenZhen",
                "O": "k8s",
                "OU": "System"
                }
            ]
        }
        EOF
        ```
    生成证书和秘钥
    ```
    cd /opt/ssl
    cfssl gencert --initca=true csr.json | cfssljson --bare ca

    # 生成证书
    for targetName in kubernetes admin kube-proxy; do
          cfssl gencert --ca  ca.pem --ca-key ca-key.pem --config config.json --profile kubernetes $targetName-csr.json | cfssljson --bare $targetName
    done
    ```
    6. kubelet 证书和私钥，生成高级审核配置文件
       kubelet 其实也可以手动通过CA来进行签发，但是这只能针对少数机器，毕竟我们在进行证书签发的时候，是需要绑定对应Node的IP的，如果node太多了，加IP就会很幸苦， 所以这里我们使用TLS 认证，由apiserver自动给符合条件的node签发证书，允许节点加入集群。

       **在kubernetes-console生成token并分发发所有的master节点**
       ```
        cd /opt/ssl
        export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
        echo "Tokne: ${BOOTSTRAP_TOKEN}"
        cat > token.csv <<EOF
        ${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
        EOF
        #生成高级审核配置文件
        cat >> audit-policy.yaml <<EOF
        # Log all requests at the Metadata level.
        apiVersion: audit.k8s.io/v1beta1
        kind: Policy
        rules:
        - level: Metadata
        EOF
       ``` 
    7. 颁发证书
    ```
    cd /opt/ssl

    for IP in `seq 151 153`;do
        ssh root@172.30.11.$IP mkdir -p /etc/kubernetes/ssl
    scp * root@172.30.11.$IP:/etc/kubernetes/ssl/
    done
    ```


#### 三、创建ETCD集群
    在k8s-master-01~03上
    ```
    cat <<EOF> /usr/lib/systemd/system/etcd.service
    [Unit]
    Description=Etcd Server
    After=network.target
    After=network-online.target
    Wants=network-online.target
    Documentation=https://github.com/coreos
    [Service]
    Type=notify
    WorkingDirectory=/var/lib/etcd/
    EnvironmentFile=-/etc/etcd/etcd.conf
    ExecStart=/usr/local/bin/etcd \
    --name=etcd01 \
    --cert-file=/etc/kubernetes/ssl/etcd.pem \
    --key-file=/etc/kubernetes/ssl/etcd-key.pem \
    --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
    --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
    --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --initial-advertise-peer-urls=https://192.168.233.201:2380 \
    --listen-peer-urls=https://192.168.233.201:2380 \
    --listen-client-urls=https://192.168.233.201:2379,http://127.0.0.1:2379 \
    --advertise-client-urls=https://192.168.233.201:2379 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=etcd01=https://192.168.233.201:2380,etcd02=https://192.168.233.202:2380,etcd03=https://192.168.233.203:2380 \
    --initial-cluster-state=new \
    --data-dir=/var/lib/etcd
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536
    [Install]
    WantedBy=multi-user.target
    EOF
    ```

#### 四、安装kube并初始化
```
rpm --import https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
cat <<EOF> /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
EOF

yum install -y kubelet-1.9.2 kubeadm-1.9.2 kubectl-1.9.2
```
修改systemd为cgroupfs(为了和docker的group一致)，并设置pause的镜像地址 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf，并新添加一行：
```
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
Environment="KUBELET_SWAP_ARGS=--fail-swap-on=false"
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
```
systemctl daemon-reload && systemctl start kubelet && systemctl enable kubelet

kubelet安装完成后，在启动时会报错。这个没关系，后期会在配置。

#### 五、使用kubeadm安装kubernetes
* 请修改etcd集群地址和apiserver节点IP以及VIP地址
* token是有关于kubernetes在1.5之后的权限验证功能bootstrap权限管理模式
* imageRepository是镜像下载地址，如果不添加默认是Google镜像仓库，被墙在国内很难下载
* 在执行kubeadm init --config=config.yaml之前，请注意保存etcd相关的秘钥信息，因为在kubeadm执行失败需要重新执行kubeadm reset时会导致/etc/kubernetes/pki目录清空，导致秘钥丢失
```
cat <<EOF> /etc/kubernetes/config.yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://192.168.233.201:2379
  - https://192.168.233.202:2379
  - https://192.168.233.203:2379
  caFile: /etc/kubernetes/ssl/ca.pem
  certFile: /etc/kubernetes/ssl/etcd.pem
  keyFile: /etc/kubernetes/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
kubernetesVersion: 1.9.2
api:
  advertiseAddress: "127.0.0.1"
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- k8s-master-1
- k8s-master-2
- k8s-master-3
- 192.168.233.201
- 192.168.233.202
- 192.168.233.203
- 192.168.233.200
imageRepository: "192.168.233.168/k8s"
EOF
kubeadm init --config=config.yaml
```
等待kubeadm 执行完成，将master1节点上/etc/kubernetes/pki/{ca.pem,ca-key.pem,client.pem,client-key.pem,ca-config.json}下的秘钥信息拷贝到其他的master节点的相同目录下，以及config.yaml文件也一并复制，同样执行：
```
kubeadm init --config=config.yaml
```

#### 六、Master HA配置
目前所谓的kubernetes HA 其实主要是API Server的HA，master上的其他组件，比如kube-controller-manager、kube-scheduler都是通过etcd做选举。而API Server一般有两种方式做HA；一种是多个API Server 做聚合为 VIP，另一种使用nginx反向代理，这里我们采用nginx的方式。

kube-controller-manager、kube-scheduler通过etcd选举，而且与master直接通过127.0.0.1:8443通信，而其他node，则需要在每个node上启动一个nginx，
每个nginx反代所有apiserver，node上的kubelet、kube-proxy、kubectl连接本地nginx代理端口，当nginx发现无法连接后端时会自动踢掉出问题的apiserver，
从而实现api server的HA。

**在master上的操作**
```
mkdir /etc/nginx
cd /etc/nginx
cat <<EOF>/etc/nginx/nginx-default.conf 
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    
    keepalive_timeout  65;
    
    include /etc/nginx/conf.d/*.conf;
}
stream {
        upstream apiserver {
            server 192.168.233.128:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 192.168.233.129:6443 weight=5 max_fails=3 fail_timeout=30s;
            server 192.168.233.130:6443 weight=5 max_fails=3 fail_timeout=30s;
        }
    server {
        listen 8443;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass apiserver;
    }
server {
        listen 8099;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass 127.0.0.1:8089;
    }
}
EOF
docker run -d -p 8443:8443 \
--name nginx-lb \
--restart always \
-v /etc/nginx/nginx-default.conf:/etc/nginx/nginx.conf \
daocloud.io/nginx
```

#### 八、配置keepalived VIP
在每个master节点上修改/etc/keepalived/keepalived.conf

**naster01**
```
cat <<EOF>/etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    mcast_src_ip 192.168.233.128
    virtual_router_id 99
    priority 102
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass zkonl402x1jc1qejpysckmire51xo7qz
    }
    virtual_ipaddress {
        192.168.233.134
    }
    track_script {
       chk_apiserver
    }
}
EOF
```
**master02**
```
cat <<EOF>/etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    mcast_src_ip 192.168.233.129
    virtual_router_id 99
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass zkonl402x1jc1qejpysckmire51xo7qz
    }
    virtual_ipaddress {
        192.168.233.134
    }
    track_script {
       chk_apiserver
    }
}
EOF
```
**master03**
```
cat <<EOF>/etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    mcast_src_ip 192.168.233.130
    virtual_router_id 99
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass zkonl402x1jc1qejpysckmire51xo7qz
    }
    virtual_ipaddress {
        192.168.233.134
    }
    track_script {
       chk_apiserver
    }
}
EOF

```