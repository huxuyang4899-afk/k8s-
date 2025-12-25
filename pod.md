#### ![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765421438385-b6a17ccb-3e73-486a-bf52-4a92df32b4cf.png)k8s 架构
```powershell
master 节点组件=============
kube-api-server ----公司前台
scheduler----负责进行容器的调度（XXX 容器在那个节点运行）
etcd----数据库
controller Manager 控制器-----容器出现故障时进行干预--是故障自愈还是 XXXXX
node 节点组件===========================================================
kubelet 通过调用本机的 docker 来创建容器（pod）
kube-proxy----容器 ip 不固定相当于一个代理 就有固定 ip 了
=======================================================
静态pod
主节点4核心组件 本身就是容器运行 k8s管理自己？
===============================
原因是：kubelet(是二进制程序)自动的调用docker管理主节点组件容器
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765369519395-1fb29313-a0fe-4532-a620-f2081a0f2b46.png)

| 组件 | 作用 |
| --- | --- |
| **kube-apiserver** | 集群唯一入口，负责认证、鉴权、准入控制 |
| **scheduler** | 决定 Pod 调度到哪个 Node |
| **controller-manager** | 维持集群期望状态、自愈、执行控制循环 |
| **etcd** | 分布式数据库，存储所有集群状态 |


| 组件 | 作用 |
| --- | --- |
| **kubelet** | 管理本机 Pod，与 apiserver 通信，调用 containerd |
| **kube-proxy** | 负责 Service 的虚拟 IP 和流量转发（iptables/ipvs） |


#### Kubernetes 扩展接口
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765370496720-a29a7ff1-ed27-405c-9365-e2074eeafa8b.png)

**CRI（Container Runtime Interface）容器运行时接口**

+ 出现在 Kubernetes 1.5，1.6 默认启用
+ 让 kubelet 通过 gRPC 调用任意符合 CRI 的容器运行时
+ 代表运行时：containerd、CRI-O
+ 作用：**解耦运行时，不再绑定 Docker**

**2. CNI（Container Network Interface）容器网络接口**

+ kubelet 在创建 Pod 时会调用 CNI 插件配置网络
+ Kubernetes 自身不提供网络实现，只提供接口
+ 代表方案：Calico、Flannel、Cilium、Weave Net

**3. CSI（Container Storage Interface）容器存储接口**

+ 标准化存储卷插件
+ Kubernetes 本身不强制依赖，但生产环境通常使用
+ 代表方案：Rook-Ceph、OpenEBS、nfs-csi、各云厂商的盘插件

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371150105-c86ec449-4f84-4daf-a8db-5c4a1fb80732.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371158870-7ac3771b-fd73-4a73-b3fa-b73ce472d574.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371165377-bd615db6-3f9a-4b44-8cba-9b02229183fd.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371269249-34f1298f-da0c-424f-b75c-6a398527432e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371604444-e5f80fe7-755d-423b-aea1-c682eef01886.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765371117436-b7a7b301-1cdb-4254-893f-7f52a9d98735.png)

#### 基于Kubeadm 和 Containerd 部署 Kubernetes 高可用集群
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765372478559-e33c771c-f735-46a4-bc57-d4296aa690b3.png)

```bash
# 修改内核参数
[root@ha1 ~]#cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_nonlocal_bind = 1
EOF
[root@ha1 ~]#sysctl -p
#安装配置haproxy
[root@ha1 ~]#apt update
[root@ha1 ~]#apt -y install haproxy
# vim /etc/haproxy/haproxy.cfg
listen stats
mode http
bind 0.0.0.0:8888
stats enable
log global
stats uri /status
stats auth admin:123456
listen kubernetes-api-6443
bind 10.0.0.200:6443
mode tcp
server master1 10.0.0.201:6443 check inter 3s fall 3 rise 3
#先暂时禁用master2和master3，等kubernetes安装完成后，再启用
server master2 10.0.0.202:6443 check inter 3s fall 3 rise 3
server master3 10.0.0.203:6443 check inter 3s fall 3 rise 3
两个haproxy节点配置文件一致即可
[root@ha1 ~]#systemctl restart haproxy.service
```

```bash
# 安装keepalived 实现 HAProxy 的高可用
[root@ha1 ~]# apt update
[root@ha1 ~]# apt -y install keepalived
[root@ha1 ~]#cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
notification_email {
acassen
}
notification_email_from Alexandre.Cassen@firewall.loc
smtp_server 192.168.200.1
smtp_connect_timeout 30
router_id ha1.wang.org # 指定router_id,#在ha2上为ha2.wang.org
}
vrrp_script check_haproxy {
#script "/etc/keepalived/check_haproxy.sh"
script "killall -0 haproxy"
interval 1
weight -30
fall 3
rise 2
timeout 2
}
vrrp_instance VI_1 {
state MASTER # 在ha2上为BACKUP
interface eth0
garp_master_delay 10
smtp_alert
virtual_router_id 66 # 指定虚拟路由器ID,ha1和ha2此值必须相同
priority 100 # 在ha2上为80
advert_int 1
authentication {
auth_type PASS
auth_pass 123456 # 指定验证密码,ha1和ha2此值必须相同
}
virtual_ipaddress {
10.0.0.200/24 dev eth0 label eth0:1 # 指定VIP,ha1和ha2此值必须相同
}
track_script {
check_haproxy
}
}
[root@ha1 ~]#systemctl restart keepalived.service
```

```bash
ssh-keygen
ssh-copy-id 127.0.0.1
for i in {202..206};do scp -r /root/.ssh 10.0.0.$i:;done
```

```bash
hostnamectl set-hostname master1.wang.org
vim /etc/hosts
10.0.0.200 kubeapi.wang.org kubeapi
10.0.0.201 master1.wang.org master1
10.0.0.202 master2.wang.org master2
10.0.0.203 master3.wang.org master3
10.0.0.204 node1.wang.org node1
10.0.0.205 node2.wang.org node2
10.0.0.206 node3.wang.org node3
172.18.0.253 harbor.wang.org harbor
for i in {202..206} 107 108 ; do scp /etc/hosts  10.0.0.$i:/etc/hosts;done
```

```bash
# 禁用swap
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
# 或者
systemctl disable --now swap.img.swap;systemctl mask swap.target

# 时间同步（可选）
# 借助于chronyd服务（程序包名称chrony）设定各节点时间精确同步
apt -y install chrony
chronyc sources -v
# 禁用默认配置的iptables防火墙服务
ufw disable
ufw status
```

如果是安装 Docker 会自动配置以下的内核参数，而无需手动实现

但是如果安装Contanerd，还需手动配置

允许 iptables 检查桥接流量,若要显式加载此模块，需运行 modprobe br_netfilter

为了让 Linux 节点的 iptables 能够正确查看桥接流量，还需要确认net.bridge.bridge-nf-call-iptables

设置为 1

```bash
# 加载模块
modprobe overlay
modprobe br_netfilter
lsmod |grep -E 'overlay|br_netfilter'
br_netfilter 32768 0
bridge 421888 1 br_netfilter
overlay 212992 0
# 开机加载
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
# 应用 sysctl 参数生效而不重新启动
sysctl --system
```

###### 所有主机安装 Containerd
```bash
# 在所有节点上安装 Containerd
[root@node1 ~]# apt update && apt -y install containerd
# 修改containerd配置基于toml(Tom's Obvious Minimal Language)格式：toml.io
[root@node1 ~]# mkdir /etc/containerd/
[root@node1 ~]# containerd config default > /etc/containerd/config.toml
#1）将 sandbox_image 镜像源设置为阿里云google_containers镜像源（国内网络需要）
[root@node1 ~]#grep sandbox_image /etc/containerd/config.toml
# 新版k8s-v1.34.2
[root@node1 ~]#sed -i "s#registry.k8s.io/pause:3.8#registry.aliyuncs.com/google_containers/pause:3.10.1#g" /etc/containerd/config.toml

#2）配置containerd cgroup 驱动程序systemd,ubuntu22.04/24.04较新内核必须修改,ubuntu20.04更旧的内核版本可不做修改
[root@node1 ~]#sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
[root@node1 ~]#grep SystemdCgroup /etc/containerd/config.toml
SystemdCgroup = true


# 镜像加速(建议，可选)
[root@node1 ~]#vim /etc/containerd/config.toml
# 在此行下面加行
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
############################添加下面#号之间的行，实现镜像加速###############################
# 新版，国内无法访问docker官方镜像，需要配置代理，# 第170行
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://docker.m.daocloud.io","https://docker.1panel.live"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.wang.org"]
  endpoint = ["https://harbor.wang.org"]
[plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.wang.org".tls]
  insecure_skip_verify = true
[plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.wang.org".auth]
  username = "admin"
  password = "123456"
######################################################################################
[root@master1 ~]# for i in {202..206};do scp -r /etc/containerd/  10.0.0.$i:/etc/containerd;done
```

###### 所有主机安装 kubeadm、kubelet 和 kubectl
```bash
# 阿里云镜像
https://mirrors.tuna.tsinghua.edu.cn/help/kubernetes/


apt-get update && apt-get install -y apt-transport-https
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.34/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.34/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl

# 安装指定版本
K8S_RELEASE_VERSION=1.34.1
apt install -y kubeadm=${K8S_RELEASE_VERSION}-1.1 kubelet=${K8S_RELEASE_VERSION}-1.1 kubectl=${K8S_RELEASE_VERSION}-1.1
```

###### 在第一个 master 节点初始化 Kubernetes 集群
```bash
# 新版
[root@master1 ~]# K8S_RELEASE_VERSION=1.34.1   与安装版本一致

kubeadm init --kubernetes-version=v${K8S_RELEASE_VERSION} --control-plane-endpoint="kubeapi.wang.org"   --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0  --image-repository registry.aliyuncs.com/google_containers  --upload-certs
```

还可以基于配置文件初始化 步骤见（3.4.7.2.2 基于文件初始化kubernetes集群生产环境推荐本方式，利用文件可以了解以前创建的配置）

###### 在第一个 master 节点生成 kubectl 命令的授权文件
kubectl是kube-apiserver的命令行客户端程序，实现了除系统部署之外的几乎全部的管理操作，是

kubernetes管理员使用最多的命令之一。kubectl需经由API server认证及授权后方能执行相应的管理操

作，kubeadm部署的集群为其生成了一个具有管理员权限的认证配置文

件/etc/kubernetes/admin.conf，它可由kubectl通过默认的“$HOME/.kube/config”的路径进行加载。当然，用户也可在kubectl命令上使用--kubeconfig选项指定一个别的位置。

下面复制认证为Kubernetes系统管理员的配置文件至目标用户（例如当前用户root）的家目录下

```bash
[root@master1 ~]#mkdir -p $HOME/.kube
[root@master1 ~]#sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```bash
  kubeadm join kubeapi.wang.org:6443 --token f8665j.4kk53cak67o28ntg \
        --discovery-token-ca-cert-hash sha256:26ed204643d61e5a0226fbd89a408f1a9991645f815b8a7b2402bd7f4494a85b \
        --control-plane --certificate-key 59e8630bf6f83096ecfc8f913eccaa526a283fe7f96ce7099da44a3ac827daf8
```

```bash
kubeadm join kubeapi.wang.org:6443 --token f8665j.4kk53cak67o28ntg \
        --discovery-token-ca-cert-hash sha256:26ed204643d61e5a0226fbd89a408f1a9991645f815b8a7b2402bd7f4494a85b
```

```bash
# 下载yaml文件
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# kubectl apply -f kube-flannel.yml -----拉取镜像因为网络原因失败
因为执行过（kubectl apply -f kube-flannel.yml）会有镜像的残留导致
[root@master1 ~]#kubectl get pods -n kube-flannel -o wide 
NAME                    READY   STATUS   RESTARTS AGE       IP           NODE    NOMINATED NODE     READINESS GATES
kube-flannel-ds-9dldq    0/1   Terminating 0        25m 10.0.0.204 node1.wang.org   <none>              <none>

# 把flannel阻塞的容器强制清理
kubectl get pods -n kube-flannel -o name | xargs -n 1 kubectl delete -n kube-flannel --grace-period=0 --force

# 从docker的环境内把当时的镜像给打包到本地了导入所有主从节点
# 在每个节点执行
ctr -n k8s.io images import flannel-k8s.tar
kubectl apply -f kube-flannel.yml
# 再次查看状态
kubectl get pods -n kube-flannel -o wide --watch
```

#### Kubeasz 利用 Ansible 部署二进制 Kubernetes 高可用集群
github链接：[https://github.com/easzlab/kubeasz](https://github.com/easzlab/kubeasz)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765421446777-915d9754-c388-4b4b-aeb5-45551d934ce5.png)

|      IP         |          主机名             |           角色                                             |

| 10.0.0.201 | master1.wang.org   | K8s 集群主节点 1，K8s 集群 etcd 节点 1，建议内存 2G 以上 |

| 10.0.0.202 | master2.wang.org   | K8s 集群主节点 2，K8s 集群 etcd 节点 2，建议内存 2G 以上 |

| 10.0.0.203 | master3.wang.org   | K8s 集群主节点 3，K8s 集群 etcd 节点 3，建议内存 2G 以上 |

| 10.0.0.204 | node1.wang.org     | K8s 集群工作节点 1                                |

| 10.0.0.205 | node2.wang.org     | K8s 集群工作节点 2                                |

| 10.0.0.206 | node3.wang.org     | K8s 集群工作节点 3                                |

10.0.0.200 作为部署 k8s 节点执行 ansible 命令

```bash
ssh-keygen
ssh-copy-id 127.0.0.1
for i in {201..206};do scp -r /root/.ssh 10.0.0.$i:;done
```

```bash
# 下载工具脚本ezdown
[root@ubuntu2404 ~]# export release=3.6.8 #kubernetes-v1.34.2 2025-12-10 成功
[root@ubuntu2204 ~]# wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
wget https://githubfast.com/easzlab/kubeasz/releases/download/${release}/ezdown

# 添加权限
[root@ubuntu2204 ~]# chmod +x ./ezdown
# 查看用法
[root@ubuntu2204 ~]# ./ezdown
Usage: ezdown [options] [args]
option:
-C stop&clean all local containers
-D download default binaries/images into "/etc/kubeasz"
-P <OS> download system packages of the OS (ubuntu_22,debian_11,...)
-R download Registry(harbor) offline installer
-S start kubeasz in a container
-X <opt> download extra images
-d <ver> set docker-ce version, default "24.0.7"
-e <ver> set kubeasz-ext-bin version, default "1.9.0"
-k <ver> set kubeasz-k8s-bin version, default "v1.29.0"
-m <str> set docker registry mirrors, default "CN"(used in Mainland,China)
-z <ver> set kubeasz version, default "3.6.3"
# 下载kubeasz代码、二进制、默认下载容器镜像到/etc/kubeasz目录并同时安装Docker，（更多关于ezdown的参数，运行./ezdown 查看）
[root@ubuntu2204 ~]# ./ezdown -D
# 默认使用清华源，如果禁用可以修改指向其它源，如阿里
[root@ubuntu2404 ~]#./ezdown -D
2025-12-10 11:52:36 [ezdown:786] INFO Action begin: download_all
2025-12-10 11:52:36 [ezdown:173] INFO downloading docker binaries, arch:x86_64,version:28.0.4
--2025-12-10 11:52:36-- https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/static/stable/x86_64/docker-28.0.4.tgz
Resolving mirrors.tuna.tsinghua.edu.cn (mirrors.tuna.tsinghua.edu.cn)...101.6.15.130, 2402:f000:1:400::2
Connecting to mirrors.tuna.tsinghua.edu.cn (mirrors.tuna.tsinghua.edu.cn)|101.6.15.130|:443... connected.
HTTP request sent, awaiting response... 403 Forbidden
2025-12-10 11:52:36 ERROR 403: Forbidden.
2025-12-10 11:52:36 [ezdown:175] ERROR downloading docker failed
[root@ubuntu2404 ~]# grep mirrors.tuna.tsinghua.edu.cn ezdown
DOCKER_URL="https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/static/stable/${ARCH}/docker-${DOCKER_VER}.tgz"
[root@ubuntu2404 ~]# sed -i "s/mirrors.tuna.tsinghua.edu.cn/mirrors.aliyun.com/" ezdown
[root@ubuntu2404 ~]# ./ezdown -D
#上述脚本运行成功后，所有文件（kubeasz代码、二进制、离线镜像）均已整理好放入目录/etc/kubeasz
## 3.6.8默认已经做docker-hub官方代理配置，无需修改
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765423780321-51ff314c-3ead-45e3-9bf4-f85bfabc7430.png)

```bash
[root@ubuntu2404 ~]#ls /etc/kubeasz/
ansible.cfg  bin  docs  down  example  ezctl  ezdown  manifests  pics  playbooks  README.md  roles  tools
# 容器化运行kubeasz，用于安装k8s集群工具
[root@ubuntu2404 ~]# ./ezdown -S
[root@ubuntu2404 ~]#docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS         PORTS     NAMES
2278fabb8268   easzlab/kubeasz:3.6.8   "tail -f /dev/null"      7 seconds ago   Up 6 seconds             kubeasz
32ab893ad8ca   registry:2              "/entrypoint.sh /etc…"   6 minutes ago   Up 6 minutes             local_registry

# 自动生成别名
[root@ubuntu2204 ~]# tail -n1 .bashrc
alias dk='docker exec -it kubeasz' # generated by kubeasz
# 创建新集群 k8s-01
[root@ubuntu2404 ~]# dk ezctl new k8s-01
2025-12-11 11:38:54 [ezctl:145] DEBUG generate custom cluster files in /etc/kubeasz/clusters/k8s-01
2025-12-11 11:38:54 [ezctl:151] DEBUG set versions
2025-12-11 11:38:54 [ezctl:182] DEBUG cluster k8s-01: files successfully created.
2025-12-11 11:38:54 [ezctl:183] INFO next steps 1: to config '/etc/kubeasz/clusters/k8s-01/hosts'
2025-12-11 11:38:54 [ezctl:184] INFO next steps 2: to config '/etc/kubeasz/clusters/k8s-01/config.yml'

# 按规划修改配置
[root@ubuntu2204 ~]#vim /etc/kubeasz/clusters/k8s-mycluster-01/hosts
[etcd]
10.0.0.201
10.0.0.202
10.0.0.203
[kube_master]
10.0.0.201 k8s_nodename='master-01' # 修改此处三行
10.0.0.202 k8s_nodename='master-02'
10.0.0.203 k8s_nodename='master-03'
[kube_node]
10.0.0.204 k8s_nodename='worker-01' # 修改此处三行
10.0.0.205 k8s_nodename='worker-02'
10.0.0.206 k8s_nodename='worker-03'
# 创建集群
# 方法1:一键安装，等价于执行docker exec -it kubeasz ezctl setup k8s-mycluster-01 all
# all 相当于01 02 ...... 07
[root@ubuntu2204 ~]# dk ezctl setup k8s-01 all
耐心等待
# 以下服务都是二进制服务
[root@master-01 ~]#systemctl status kubelet.service kube-apiserver.service kube-controller-manager.service kube-scheduler.service etcd.service kube-proxy.service
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765451356122-c577434a-25f2-4106-8d7c-11b1a08731b8.png)

安装后发现 master 节点 6443 端口有本地 的 10.0.0.201 的还有 127.0.0.1 的

而 worker 节点只有 127.0.0.1 的 6443 他是 kube-lb 的负载均衡器的 6443

```bash
[root@worker-01 ~]#ls /proc/3969
arch_status  comm                fd                 limits     mountstats     pagemap      sessionid     status          wchan
attr         coredump_filter     fdinfo             loginuid   net            patch_state  setgroups     syscall
autogroup    cpu_resctrl_groups  gid_map            map_files  ns             personality  smaps         task
auxv         cpuset              io                 maps       numa_maps      projid_map   smaps_rollup  timens_offsets
cgroup       cwd                 ksm_merging_pages  mem        oom_adj        root         stack         timers
clear_refs   environ             ksm_stat           mountinfo  oom_score      sched        stat          timerslack_ns
cmdline      exe                 latency            mounts     oom_score_adj  schedstat    statm         uid_map
[root@worker-01 ~]#ll /proc/3969/exe
lrwxrwxrwx 1 root root 0 Dec 11 19:23 /proc/3969/exe -> /etc/kube-lb/sbin/kube-lb*
[root@worker-01 ~]#ls /etc/kube-lb/
conf  logs  sbin
就是nginx的配置文件；做了负载均衡 调度到master节点 
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765452392797-160fc808-514b-41b3-b560-7b2ef6f5529b.png)

#### 脚本安装 1 主 3 从
准备 cri-dockerd 对应版本的包--还有脚本的 ip 地址要规划好-----k8s 的版本

安装好后从节点执行加入集群

最后

```bash
# 主节点下载文件
wget https://githubfast.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# 所有节点导入flannel镜像
docker load -i flannel-k8s.tar
# 主节点执行
[root@master1 ~]# kubectl apply -f kube-flannel.yml
[root@master1 ~]# kubectl get nodes
NAME               STATUS   ROLES           AGE     VERSION
master1.wang.org   Ready    control-plane   7m21s   v1.34.1
node1.wang.org     Ready    <none>          6m36s   v1.34.1
node2.wang.org     Ready    <none>          6m28s   v1.34.1
node3.wang.org     Ready    <none>          6m21s   v1.34.1
```

#### Kubernetes 资源对象和 Pod 资源
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765456224201-18f5c722-6134-4f08-bf32-11f644901f1d.png)

```bash
# 查看资源类型
[root@master1 ~]# kubectl api-resources
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765456828767-591c346c-2909-437a-ae61-fd83de2cd1d8.png)

V1 是最早期的资源 ；后面太多了为了便于管理 增加组织名字/版本号

KIND 字段是某个资源写进配置文件时应用的资源名字

NAMESPACED 字段是 true 的资源类型就是名称空间的资源；false 就是集群的资源类型

名称空间的只能在某个固定的名称空间内只有，集群的就是可以在不同的名称空间共用

同一资源在 同一名称空间 内 名称不同 也是可以的

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765459992475-e49833a7-9ba9-46c6-a76d-7ebbebf0c63e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765460007050-642449a6-ae9b-487b-a98b-7dd26cae2ce7.png)

```bash
kubectl get svc -A 是用来 查看集群中所有命名空间的 Service 资源 的
kubectl get XXX  -A  查看所有命名空间中XX资源

# 查询 default 命名空间下的所有svc类型的资源
[root@master1 ~]# kubectl get svc -n default 
# 查询 default 命名空间下的所有 Svc的类型资源
[root@master1 ~]# kubectl get svc -n default -o yaml
# 查询 default 命名空间下名字为 kubernetes 的 Svc的类型资源
[root@master1 ~]#kubectl get svc -n default  kubernetes -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2025-12-11T12:11:48Z"
  labels:
    component: apiserver
    provider: kubernetes
  name: kubernetes
  namespace: default
  resourceVersion: "243"
  uid: 9bf26200-6b78-466c-81c1-25726a98da6b
spec:
  clusterIP: 10.96.0.1
  clusterIPs:
  - 10.96.0.1
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 6443
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
5个字段
```

```bash
# 查看资源类型文档
kubectl explain pod
kubectl explain pod.spec
```

##### 名称空间 namespaces
Kubernetes 的资源工作的有效范围分成两种级别:

集群级别: 针对整个Kubernetes集群内都有效

名称空间级别: 只针对指定名称空间内有效,而不属于任务名称空间

名称空间 Namespace 用于将集群分隔为多个隔离的逻辑分区以配置给不同的用户、租户、环境或者项

目使用。

| 类型 | NAMESPACED | 作用域 | 示例 |
| --- | --- | --- | --- |
| 命名空间资源（Namespaced Resource） | true | 只在某个命名空间内有效 | Pod、Deployment、Service、ConfigMap |
| 集群资源（Cluster Resource） | false | 在整个集群范围内有效，不属于任何命名空间 | Node、Namespace、PersistentVolume、ClusterRole |


##### 资源创建方法
```bash
# 指令法
[root@master1 ~]# kubectl create --help
# 创建名称空间
kubectl create NAME
# 配置文件法

# 查看XX名称空间
kubectl get ns default -o yaml 
参考改内容

指令式对象配置,没有幂等性
kubectl create -f xxx.yaml
声明式对象配置,有幂等性，个别资源没有
kubectl apply -f xxx.yaml
```

```bash
[root@master1 ~]# kubectl create ns m66 -o yaml --dry-run=client > ns-m66.yml
[root@master1 ~]#cat ns-m66.yml
apiVersion: v1
kind: Namespace
metadata:
  name: m66
spec: {}
status: {}
模拟执行 可以生成默认的清单文件

# 删除名称空间
[root@master1 ~]#kubectl delete ns m66
namespace "m66" deleted
如何名称空间有资源 那么 将会被一起删除
```

kubernetes 和Docker的名称空间的区别

kubernetes的名称空间, 用户根据需要可以自行创建删除,属于用户级别

Docker的名称空间是内核内置,用户不能创建删除,属于内核级别

##### Pod
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765462680075-66a71d2a-d69d-426c-9deb-ef76dff67c0a.png)

Pod 的组成形式有两种

单容器Pod：除Pause容器外,仅含有一个容器

多容器Pod：除Pause容器外，含有多个具有“超亲密”关系的容器，一般由主容器和辅助容器（比

如：sidecar容器）构成

```bash
Pod ： 业务容器 （老板容器）+ Pause 容器附加（小秘书容器）+ 容器（sidecar 边车容器，高级秘书容器，高级定制功能，网格），多个亲密关系的容器的集合，相当于一个家庭

- 容器网络 172.17.0.X/16
- 容器存储 

命名空间（Namespace）是 逻辑隔离 的单位。

主要作用是把集群里的资源分类管理，不同命名空间的资源相互隔离。

命名空间里可以存放的资源Pod、Service、ConfigMap、Secret、Deployment、StatefulSet、DaemonSet 等 几乎所有可命名的资源（即 NAMESPACED=true 的资源类型）。

这些资源在同一命名空间内名称唯一，但不同命名空间可以有相同名称的资源。

例子

Namespace default：Pod nginx, Service nginx-svc

Namespace test：也可以有 Pod nginx, Service nginx-svc，互不影响

非命名空间资源

比如 ClusterRole、Node 等，NAMESPACED=false，不属于任何命名空间，全局共享。

总结：命名空间就是资源的容器，一个命名空间可以存放多种类型的资源，实现逻辑隔离和管理。
```

```bash
Pod 资源分类：
自主式 Pod
由用户直接定义并提交给API Server创建的Pods
节点故障后不具有自动调度至其它节点的故障自愈能力
---------------------------------------------------------
由Workload Controller 管控的 Pod
比如: 由Deployment控制器（controller Manager）管理的Pod
节点故障后具有自动调度至其它节点的故障自愈能力
---------------------------------------------------------
静态 Pod
由kubelet加载配置信息后，自动在对应的节点上创建的Pod
用于实现Master节点上的系统组件API Server 、Controller-Manager 、Scheduler 和Etcd功能的Pod
相关配置存放在控制节点的 /etc/kubernetes/manifests 目录下
```

###### 自主式 Pod
自主式Pod是由Pod本身去控制本身，不受其它资源如控制器控制的 Pod

这种Pod本身**不具有自我修复功能**，当自主式Pod被创建后，都会被Kubernetes调度到集群的Node上。

不论是否故障,都会在此Node节点上存在,直到Pod的进程终止和被删掉

中间如果Pod即使出了故障, Pod也不会被调度至其它节点实现自愈。

```bash
# 可以支持三种方法创建自主式pod

# 指令式命令
kubectl run NAME --image=image [--port=port] [--replicas=replicas]
kubectl run NAME --image=image [--env="key=value"] [--port=port] [--dry-run=server|client] [--overrides=inline-json] [--command] -- [COMMAND] [args...][options]

# 初始化一个Pod对象，包含一个nginx容器
[root@master1 ~]# kubectl run nginx01 --image=registry.cn-beijing.aliyuncs.com/wangxiaochun/nginx:1.26.0

# 指令式对象配置文件
[root@master1 ~]# kubectl run nginx01 --image=registry.cn-beijing.aliyuncs.com/wangxiaochun/nginx:1.26.0 --dry-run=client -o yaml > nginx.yaml
# 更改配置文件
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx02
  namespace: m65
spec:
  containers:
  - name: nginx
    image: nginx:1.26.0
# kubectl create -f nginx.yaml

# 查看pod在哪个node节点
# kubectl get pod -o wide
还可以指定名称空间创建pod
那么他自己分配node节点 那么名称空间也会在对应的node节点创建嘛？
✅ 总结：Namespace 是逻辑隔离，不占节点；Pod 才是会被 Scheduler 调度到具体 Node 上的工作负载。
不存在的 Namespace 不会自动创建
# 声明式对象配置文件
# kubectl apply -f pod.yaml  与指令式对象配置文件用一样的配置文件 有幂等性
```

##### 删除 Pod
```bash
# 优雅删除
[root@master1 ~]#kubectl delete pod pod-test1
# 立即删除
[root@master1 ~]#kubectl delete pod pod-test1 --force --grace-period=0
# 删除demo名称空间的所有Pod
[root@master1 ~]#kubectl delete pod --all -n demo
# 删除所有pod
[root@master1 ~]#kubectl get po|cut -d" " -f1 | tail -n +2 |xargs kubectl delete pod
# 快速删除所有pod
[root@master1 ~]#kubectl get pod|awk 'NR!=1{print $1}'|xargs -i kubectl delete pod {} --force --grace-period=0
```

##### pod 状态
pending-----网络插件未安装 

Errimagepull----镜像拉取失败

ImagePullBackOff-----再次镜像拉取失败

ContainerCreating**-----**正在创建，拉镜像

```bash
[root@master1 ~]#kubectl get pod
NAME      READY   STATUS             RESTARTS   AGE
nginx01   0/1     ImagePullBackOff   0          12s
# 详细查看报错原因
[root@master1 ~]#kubectl get pod nginx01 -o yaml

state:
      waiting:
        message: 'Back-off pulling image "registry.cn-beijing.aliyuncs.com/wangxiaochun/nginx:1.28":
          ErrImagePull: Error response from daemon: manifest for registry.cn-beijing.aliyuncs.com/wangxiaochun/nginx:1.28
          not found: manifest unknown: manifest unknown'
          
# kubectl describe pod <pod名字>
[root@master1 ~]# kubectl describe pod nginx01
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765545412357-15a3ad32-5d54-4bb3-a5ed-142ad5312a7c.png)

```bash
[root@master1 ~]#kubectl get pod
NAME      READY   STATUS             RESTARTS   AGE
nginx01   0/1     ImagePullBackOff   0          5m45s

第二次拉取又失败

# 查看更加详细的报错信息
[root@master1 ~]#kubectl logs mynginx2（所在的pod名字） mynginx（容器名字）
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765546242000-3da77b23-3b82-4ba0-81c7-b36984b4d95e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765546313092-71c8208b-84a1-4d08-ab0d-1e9a75cf3833.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765546330434-e0906691-0b77-49b7-9c53-57f86649fc9f.png)

```bash
[root@master1 ~]# kubectl run mysql --env="TZ=Asia/Shanghai" --env="MYSQL_RANDOM_ROOT_PASSWORD=yes" --image registry.cn-beijing.aliyuncs.com/wangxiaochun/mysql:8.0.29-oracle
# 因为是随机密码 所以去日志查看
kubectl logs mysql
# 进入容器
kubectl exec -it mysql --sh 
```

#####  利用command和args字段传递容器的启动命令和参数
command ：对应覆盖 Docker 中的 `ENTRYPOINT``

`args ：对应覆盖 Docker 中的 CMD

```bash
containers:
- name: redis
  image: redis
  command: ["redis-server"] # 覆盖 ENTRYPOINT
  args: ["--port", "6380"] # 覆盖 CMD

  这相当于在 Docker 中执行：
  docker run --entrypoint "redis-server" redis --port 6380
```

```yaml
#用自定义的命令替换sh
[root@master1 ~]# cat pod-with-cmd-admin-box.yaml
apiVersion: v1
kind: Pod
metadata:
  name: admin-box
spec:
  ontainers:
  - image: registry.cn-beijing.aliyuncs.com/wangxiaochun/busybox:1.32.0
    #image: registry.cn-beijing.aliyuncs.com/wangxiaochun/admin-box:v0.1
    name: admin-box
    command: ['sleep','inf']
    #command: ['sleep']
    #args: ['inf']
    
# 验证结果
[root@master1 ~]#kubectl exec -it admin-box -- sh
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 sleep inf
    7 root      0:00 sh
   13 root      0:00 ps
```

```yaml
command: ['sleep','inf']
    #command: ['sleep']
    #args: ['inf']
可以在command写成数组 也可以 通过command 和 args 指定
```

##### 使用宿主机网络实现容器的外部访问
默认容器使用私有的独立网段，无法从集群外直接访问，可以通过下面两种方式实现外部访问

让容器直接使用宿主机的网络地址，即容器使用host的网络模型

让容器通过宿主机的端口映射实现，即DNAT

```yaml
# 清单文件
[root@master1 ~]# cat pod-hostnetwork.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostnetwork-demo
spec:
  #nodeName: node2.wang.org
  hostNetwork: true #直接使用容器所在宿主机网络
  containers:
  - name: demo-env
    image: registry.cn-beijing.aliyuncs.com/wangxiaochun/pod-test:v0.1
    env:
    - name: PORT
      value: "9999" #要确保此端口在宿主机不能被占用
[root@master1 ~]# kubectl apply -f pod-hostnetwork.yaml
```

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765548927380-a06d5e3b-575c-4031-85d6-c05ddeadd106.png)

##### 使用容器所在宿主机指定的端口
```yaml
[root@master1 ~]#vim pod-hostport.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostport-demo
spec:
  containers:
  - name: demo-env
    image: registry.cn-beijing.aliyuncs.com/wangxiaochun/pod-test:v0.1
    env:
    - name: PORT             #镜像的环境变量可以指定容器运行时使用的端口
      value: "9999"
    ports:
    - name: http             #不支持大写字母
      containerPort: 9999    #使用上面变量相同的端口
      hostPort: 8888         #使用宿主机的指定端口
```

#### Pod 创建的启动流程
![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765549371711-2f3d9c22-2296-4f51-b26b-82ad785cd06c.png)

![](https://cdn.nlark.com/yuque/0/2025/png/59496936/1765549385583-522d705b-0f98-459c-878b-7a79e0c2bc4c.png)

在 Kubernetes 中，Pod 的创建流程是这样的：用户通过 kubectl 提交 Pod，API Server 会进行权限与准入校验，然后将 Pod 的期望状态写入 etcd。Scheduler 通过 watch API Server 发现未调度的 Pod，完成节点选择后把调度结果通过 binding API 写回 API Server。API Server 更新 etcd 后，目标节点上的kubelet 通过 watch 发现有新 Pod 分配到本节点，然后调用 container runtime （容器运行时）拉取镜像、创建并启动容器。容器启动后，kubelet 会周期性上报实际状态给 API Server，最终 API Server 再写入 etcd，使整个集群达到期望状态。

 

