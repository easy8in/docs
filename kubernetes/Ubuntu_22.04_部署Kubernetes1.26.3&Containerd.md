# Ubuntu 22.04 Live Server

## 系统要求

1. Kubernetes 项目为基于 Debian 和 Red Hat 的 Linux 发行版以及一些不提供包管理器的发行版提供通用的指令
2. 每台机器 2 GB 或更多的 RAM （如果少于这个数字将会影响你应用的运行内存)
3. 2 CPU 核或更多
4. 集群中的所有机器的网络彼此均能相互连接 (公网和内网都可以)
5. 节点之中不可以有重复的主机名、MAC 地址或 product_uuid。请参见这里了解更多详细信息。
6. 禁用交换分区。为了保证 kubelet 正常工作，你 必须 禁用交换分区。

## Kubernetes master 资源需求 (生产环境)

Kubernetes Master 节点的配置取决于 Kubernetes 集群中 Node 节点个数，节点数越多，需要的资源也就越多。节点数可根据需要做微调。

| Kubernetes 集群 Node 节点个数 | Kubernetes Master 节点配置 |
| ----------------------------- | -------------------------- |
| 1-5                           | 1vCPUs 4GB Memory          |
| 6-10                          | 2vCPUs 8GB Memory          |
| 11-100                        | 4vCPUs 16GB Memory         |
| 101-250                       | 8vCPUs 32GB Memory         |
| 251-500                       | 16vCPUs 64GB Memory        |
| 501-5000                      | 32vCPUs 128GB Memory       |

## 二进制安装文档

[参考文档](https://blog.csdn.net/qq_44078641/article/details/120049473)

[官方下载地址](https://www.downloadkubernetes.com/)

[GitHub下载地址](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/)

## 准备工作

1. 你可以使用命令 ip link 或 ifconfig -a 来获取网络接口的 MAC 地址
2. 可以使用 sudo cat /sys/class/dmi/id/product_uuid 命令对 product_uuid 校验

## linux 操作系统参数

```bash
# 配置 Irqbalance 服务
# Irqbalance 服务可以将各个设备对应的中断号分别绑定到不同的 CPU 上，以防止所有中断请求都落在同一个 CPU 上而引发性能瓶颈。
systemctl enable irqbalance
systemctl start irqbalance

# CPUfreq 调节器模式设置
# 为了让 CPU 发挥最大性能，请将 CPUfreq 调节器模式设置为 performance 模式。
cpupower frequency-set --governor performance

# Ulimit 设置
cat <<EOF >>  /etc/security/limits.conf
*        soft        nofile        1048576
*        hard        nofile        1048576
*        soft        stack         10240
EOF

sysctl --system

# 系统全局允许分配的最大文件句柄数:
sysctl -w fs.file-max=1048576
sysctl -w fs.nr_open=1048576
echo 1048576 > /proc/sys/fs/nr_open
echo 1048576 > /proc/sys/fs/file-max
 
# 允许当前会话 / 进程打开文件句柄数:
ulimit -n 1048576

# /etc/sysctl.conf
# 持久化 'fs.file-max' 设置到 /etc/sysctl.conf 文件:
fs.file-max = 1048576

# /etc/systemd/system.conf 设置服务最大文件句柄数:
DefaultLimitNOFILE=1048576
```

## TCP 协议栈网络参数

```bash
# 并发连接 backlog 设置:
sysctl -w net.core.somaxconn=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=16384
sysctl -w net.core.netdev_max_backlog=16384

# 可用知名端口范围:
sysctl -w net.ipv4.ip_local_port_range='1024 65535'

# TCP Socket 读写 Buffer 设置:
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.core.optmem_max=16777216

#sysctl -w net.ipv4.tcp_mem='16777216 16777216 16777216'
sysctl -w net.ipv4.tcp_rmem='1024 4096 16777216'
sysctl -w net.ipv4.tcp_wmem='1024 4096 16777216'

# TCP 连接追踪设置:
sysctl -w net.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

# TIME-WAIT Socket 最大数量、回收与重用设置:
sysctl -w net.ipv4.tcp_max_tw_buckets=1048576
# 注意：不建议开启該设置，NAT 模式下可能引起连接 RST
# sysctl -w net.ipv4.tcp_tw_recycle=1
# sysctl -w net.ipv4.tcp_tw_reuse=1

# FIN-WAIT-2 Socket 超时设置:
sysctl -w net.ipv4.tcp_fin_timeout=15
```

## 开始安装

```bash
# 当前采用 Ubuntu 22.04 Live Server

# 查看系统信息
lsb_release -a

# 设置超级用户密码
sudo passwd root

# 切换为超级用户
su -

# 修改 /etc/hosts 配置
hosts -> 192.168.18.201 k8s-master

# 设置主机名
hostnamectl set-hostname k8s-master

# 设置系统时区
timedatectl set-timezone Asia/Shanghai

# 激活 ufw
ufw enable

# 允许 ssh 远程访问
ufw allow 22

# 配置允许桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.core.somaxconn = 32768
vm.swappiness = 0
net.ipv4.tcp_syncookies = 0
net.ipv4.ip_forward = 1
fs.file-max = 1000000
fs.inotify.max_user_watches = 1048576
fs.inotify.max_user_instances = 1024
net.ipv4.conf.all.rp_filter = 1
net.ipv4.neigh.default.gc_thresh1 = 80000
net.ipv4.neigh.default.gc_thresh2 = 90000
net.ipv4.neigh.default.gc_thresh3 = 100000
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

# CentOS系统需要 Ubuntu 不需要
# 关闭selinux
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 关闭swap交换
swapoff -a
## 修改swap配置文件，注释掉swap相关行
# vim /etc/fstab
sed -i 's/^\(.*swap.*\)$/#\1/' /etc/fstab

# 采用 containerd 作为容器运行时
# 通过代码仓库下载 https://github.com/containerd/containerd/releases/tag/v1.7.0
# x86_64 下载 containerd-1.7.0-linux-amd64.tar.gz
# Step 1:
# 解压程序到命名目录
sudo tar Cxzvf /usr/local containerd-1.7.0-linux-amd64.tar.gz

# Step 2:
# 生成开机器启动服务，参考代码仓库配置
# https://github.com/containerd/containerd/blob/main/containerd.service
cd /etc/systemd/system/
sudo cat <<EOF >/etc/systemd/system/containerd.service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

# Step 3:
# 生成 containerd 容器配置文件 采用 systemd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# ---
# - 方式一
# sed -i 's/SystemdCgroup\ =\ false/SystemdCgroup\ =\ true/g' /etc/containerd/config.toml

# - 方式二
vim /etc/containerd/config.toml
# ---

# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#    SystemdCgroup = true

# Step 4:
# 配置镜像加速
# 方式一
# ---

[plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://xxx.mirror.aliyuncs.com", "https://registry-1.docker.io"]
# ---

# - 方式二
# ---
sed -i 's/config\_path\ =.\*/config\_path = \"\/etc\/containerd\/certs.d\"/g' /etc/containerd/config.toml

mkdir /etc/containerd/certs.d/docker.io -p

cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://sfr32v6p.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]
EOF
# ---

# ---

# Step 5:
# 修改 containerd sandbox_image
# sed -i 's/sandbox\_image\ =.\*/sandbox\_image\ =\ "registry.aliyuncs.com\/google\_containers\/pause:3.9"/g' /etc/containerd/config.toml|grep sandbox\_image"
sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9"

# Step 6:
# 启动 containerd 服务
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd
# systemctl daemon-reload && systemctl restart containerd

# Step 6:
# 安装 runc
# 下载地址 https://github.com/opencontainers/runc/releases
install -m 755 runc.amd64 /usr/local/sbin/runc
runc -v

# 安装 kubernetes
# cri-docker  cri-socket = unix:///var/run/cri-dockerd.sock
# containerd  cri-socket = unix:///run/containerd/containerd.sock

# 防火墙开放Kubernetes所需端口
# Master 开放防火墙端口
ufw allow 6443/tcp
ufw allow 2379:2380/tcp
ufw allow 10250/tcp
ufw allow 10259/tcp
ufw allow 10257/tcp

# 节点
ufw allow 10250/tcp
ufw allow 30000:32767/tcp

# 添加aliyun repo 仓库
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet=1.26.3-00 kubeadm=1.26.3-00 kubectl=1.26.3-00

# 所有安装的程序包 

# Get:1 https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial/main amd64 cri-tools amd64 1.26.0-00 [18.9 MB]
# Get:2 http://cn.archive.ubuntu.com/ubuntu jammy/main amd64 conntrack amd64 1:1.4.6-2build2 [33.5 kB]
# Get:3 http://cn.archive.ubuntu.com/ubuntu jammy/main amd64 ebtables amd64 2.0.11-4build2 [84.9 kB]
# Get:4 http://cn.archive.ubuntu.com/ubuntu jammy/main amd64 socat amd64 1.7.4.1-3ubuntu4 [349 kB]
# Get:5 https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial/main amd64 kubernetes-cni amd64 1.2.0-00 [27.6 MB]                                        
# Get:6 https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial/main amd64 kubelet amd64 1.26.3-00 [20.5 MB]                                              
# Get:7 https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial/main amd64 kubectl amd64 1.26.3-00 [10.1 MB]                                              
# Get:8 https://mirrors.tuna.tsinghua.edu.cn/kubernetes/apt kubernetes-xenial/main amd64 kubeadm amd64 1.26.3-00 [9,747 kB]

# 设置 kubelet
# --cri-socket unix:///var/run/cri-dockerd.sock
# --cgroup-driver=systemd
# 配置文件目录 /etc/systemd/system/kubelet.service.d 默认 KUBELET_CONFIG_ARGS 没有 --cgroup-driver

sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf <<-'EOF'
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
EOF

# 准备镜像文件（可选）
## 查看镜像
kubeadm config images list \
--kubernetes-version=v1.26.3 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

kubeadm config images pull \
--kubernetes-version=v1.26.3 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--cri-socket unix:///run/containerd/containerd.sock

#
# 初始化集群
# 采用containerd 运行时
kubeadm init \
--kubernetes-version=v1.26.3 \
--control-plane-endpoint=192.168.18.202 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--cri-socket unix:///run/containerd/containerd.sock \
| tee kubeadm-init.log

# 环境变量 kubectl
---
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
---

# 配置 crictl 配置
vim /etc/crictl.yml
runtime-endpoint: "unix:///var/run/containerd/containerd.sock"
image-endpoint: "unix:///var/run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false

#
# 安装网络插件 Calico
# curl https://docs.tigera.io/archive/v3.25/manifests/calico.yaml -O
# https://github.com/projectcalico/calico/blob/v3.25.0/manifests/calico.yaml
# 可以在 github 代码仓库找到
git clone https://github.com/projectcalico/calico
cp calico/manifests/calico.yaml ./
vim calico.yaml
-----

 # no effect. This should fall within `--cluster-cidr`.
 - name: CALICO_IPV4POOL_CIDR
   value: "10.244.0.0/16"
 # Disable file logging so `kubectl logs` works.

------
# 10.244.0.0/16 为初始时 --pod-network-cidr 的值
# 查看 calico 用到的镜像
cat calico.yaml | grep image:

# 拉取镜像
crictl pull docker.io/calico/cni:v3.25.0 \
&& crictl pull docker.io/calico/cni:v3.25.0 \
&& crictl pull docker.io/calico/node:v3.25.0 \
&& crictl pull docker.io/calico/node:v3.25.0 \
&& crictl pull docker.io/calico/kube-controllers:v3.25.0

# 应用网络
kubectl apply -f calico.yaml
# 查看 calico pods 启动状态
watch kubectl get pods -A

# 单机集群 让 master 可以调度pod
# 要去除污点
# 查看污点 
kubectl describe node k8s-master | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule

# 去除污点
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint node k8s-master node-role.kubernetes.io/master:NoSchedule-

# 查看 token 列表
# kubeadm token list

# 部署 deployment nginx
mkdir nginx && cd nginx

sudo tee nginx-deployment.yaml <<-'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 告知 Deployment 运行 2 个与该模板匹配的 Pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF

kubectl apply -f nginx-deployment.yaml

kubectl describe deployment nginx-deployment

kubectl get pods -l app=nginx -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
nginx-deployment-85996f8dbd-969pg   1/1     Running   0          11m   10.244.235.196   k8s-master   <none>           <none>
nginx-deployment-85996f8dbd-gwrvw   1/1     Running   0          11m   10.244.235.197   k8s-master   <none>           <none>

curl 10.244.235.196
curl 10.244.235.197

# 完成 nginx 部署

# 进度 pod bash 命令界面
kubectl exec -it nginx-deployment-85996f8dbd-r85mk -- /bin/bash
cd /usr/share/nginx/html

# 修改 hmlt 内容 验证 service 负载均衡

tee index.html <<-'EOF'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>root@nginx-deployment-85996f8dbd-lj6tm</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
EOF

# nginx-service.yaml 发布 kubernetes service
sudo tee nginx-service.yaml <<-'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
   - protocol: TCP
     port: 8888
     targetPort: 80
  selector:
    app: nginx
  #type: ClusterIP
  #type: NodePort
EOF

kubectl apply -f nginx-service.yaml

# 查看部署的服务 kubectl get services -A
kubectl get svc nginx-svc

# 验证 servcie 服务负载均衡
curl ${nginx-svc.CLUSTER-IP.port}

# 安装 ingress-nginx-controller
# https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/do/deploy.yaml

curl https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/do/deploy.yaml -o

# 查看需要的镜像
cat deploy.yaml | grep image:

# 下载镜像(替换国内镜像源)
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen
crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller

# 修改 deploy.yaml 文件
# step 1: 修改 deploy.yaml 中用到的镜像名 注释调 sha256验证
# setp 2：修改 kind: deployment 的 spec
hostNetwork: true
nodeSelector:
  ingress: nginx

# 设置部署 ingress-controller 绑定到的节点标签
kubectl label nodes k8s-master ingress=nginx

# 查看节点标签用法
kubectl label nodes k8s-master --show-labels

# 覆盖节点标签用法
kubectl label nodes k8s-master ingress=nginx --overwrite

# 删除节点标签用法 <key> -> ingress
kubectl label nodes k8s-master ingress-

# 使 ingress-nginx-controller 生效
kubectl apply -f deploy.yaml

# 发布 ingress
cat <<EOF > nginx-request-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: nginx-ingress.info
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 8888
EOF

# 发布 ingress
kubectl apply -f nginx-request-ingress.yaml

# 查看 ingress
kubectl get ingress

# 通过域名访问绑定 ingress 对应 service 的应用
curl nginx-ingress.info

---
# 安装 helm
helm-v3.11.2-linux-amd64.tar.gz
tar zxf helm-v3.11.2-linux-amd64.tar.gz
mv linux_amd/heml /usr/local/bin/

helm repo add gitlab https://charts.gitlab.io/
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/ 

# 支持自建 K8S servcie 的 LoadBalancer
# MetalLB https://github.com/danderson/metallb
# 参考 1：https://www.jianshu.com/p/55be39dccb8b
# 参考 2：https://cloud.tencent.com/developer/article/1631910
# WHY：为了从外部访问 Kubernetes 群集，目前只能使用 NodePort 或 Ingress 的方法进行服务暴露。前者的缺点是每个暴露的服务需要占用所有节点的某个端口，后者的缺点是仅仅能支持 HTTP 协议。

```