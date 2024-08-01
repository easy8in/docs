## 部署说明

1. 因为我这里是用的腾讯云服务所以需要打开对应的安全组，如果是虚拟机的话配置对应的虚拟机即可这里就不做说明了网上有
2. 建议升级内核至 4.18 版本以上否则可能会出现 [内存溢出](https://so.csdn.net/so/search?q=内存溢出&spm=1001.2101.3001.7020) 的 bug
3. k8s 在 1.24 版本剔除了 docker 做为容器运行时，因此如果想继续使用 docker 需要安装 cri-docker
4. 如果开启 ipvs 还需要安装 ipvsadm （可选）

### 软件环境

| 软件       | 版本          |
| ---------- | ------------- |
| 操作系统   | CentOS7.9_x64 |
| docker     | 20.10.22      |
| cir-docker | 0.3.0         |
| Kubernetes | 1.26.0        |
|            |               |

### 服务器

| 角色          | IP             | 组件 |
| ------------- | -------------- | ---- |
| qcloud-host01 | 1.117.115.10   |      |
| qcloud-node01 | 134.175.228.10 |      |
|               |                |      |

### 升级系统以及内核

```bash
yum update -y --exclude=kernel*

wget https://mirrors.aliyun.com/elrepo/kernel/el7/x86_64/RPMS/
rpm -ivh kernel-ml-*
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg


hostnamectl set-hostname qcloud-host01

vi /etc/hosts
1.117.115.10      qcloud-host01
134.175.228.10    qcloud-node01
```

### 优化 journald 日志

```bash
mkdir -p /var/log/journal
mkdir -p /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=1G
# 单日志文件最大 200M
SystemMaxFileSize=10M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald && systemctl enable systemd-journald
```

### 安装 ipvsadm

```bash
yum install ipvsadm ipset sysstat conntrack -y
cat >> /etc/modules-load.d/ipvs.conf <<EOF 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl restart systemd-modules-load.service
lsmod | grep -e ip_vs -e nf_conntrack

yum -y install http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.2-1.el8.x86_64.rpm

rpm -qa | grep libseccomp
```

### 转发 IPv4 并让 iptables 看到桥接流量

```bash
cat >> /etc/modules-load.d/k8s.conf <<EOF 
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter


cat >> /etc/sysctl.d/k8s.conf <<EOF 
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF


sudo sysctl --system
```

## 自签 TLS 证书

### 下载软件

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_linux_amd64 
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_linux_amd64 
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl-certinfo_1.6.3_linux_amd64
chmod +x cfssl_1.6.3_linux_amd64 cfssljson_1.6.3_linux_amd64 cfssl-certinfo_1.6.3_linux_amd64
mv cfssl_1.6.3_linux_amd64 /usr/local/bin/cfssl 
mv cfssljson_1.6.3_linux_amd64 /usr/local/bin/cfssljson 
mv cfssl-certinfo_1.6.3_linux_amd64 /usr/local/bin/cfssl-certinfo


mkdir -p /opt/certificate/etcd/conf
mkdir -p /opt/certificate/kubernetes/conf
```

### 生成 [etcd](https://so.csdn.net/so/search?q=etcd&spm=1001.2101.3001.7020) 证书

#### etcd-ca 证书

```bash
cd /opt/certificate/etcd/conf
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF
cat > ca-csr.json <<EOF
{
    "CA": {
        "expiry": "87600h"
    },
    "CN": "etcd-cluster",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "TS": "Beijing",
            "L": "Beijing",
            "O": "etcd-cluster",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### etcd 服务端证书

```bash
cat > etcd-server-csr.json << EOF
{
  "CN": "etcd-server",
  "hosts": [
    "1.117.115.10",
    "134.175.228.10",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "etcd-server",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  etcd-server-csr.json | cfssljson -bare etcd-server
```

#### etcd 客户端证书

```bash
cat > etcd-client-csr.json << EOF
{
  "CN": "etcd-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "etcd-client",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  etcd-client-csr.json | cfssljson -bare etcd-client
```

#### 拷贝证书

```bash
mv *.pem ../
```

### 生成 kubernetes 各组件证书

#### kube-ca 证书

```bash
cd /opt/certificate/kubernetes/conf
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
cat > ca-csr.json <<EOF
{
  "CA": {
    "expiry": "87600h"
  },
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "kubernetes",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

#### kube-apiserver 证书

```bash
cat > kube-apiserver-csr.json <<EOF
{
  "CN": "kube-apiserver",
  "hosts": [
    "10.0.0.1",
    "127.0.0.1",
    "1.117.115.10",
    "134.175.228.10",
    "42.192.161.108",
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
      "TS": "Beijing",
      "L": "Beijing",
      "O": "kube-apiserver",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-apiserver-csr.json | cfssljson -bare  kube-apiserver
```

#### proxy-client 证书

```bash
cat > front-proxy-ca-csr.json <<EOF
{
  "CA": {
    "expiry": "87600h"
  },
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca
cat > front-proxy-client-csr.json <<EOF
{
  "CN": "front-proxy-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert \
-ca=front-proxy-ca.pem \
-ca-key=front-proxy-ca-key.pem  \
-config=ca-config.json   \
-profile=kubernetes front-proxy-client-csr.json | cfssljson -bare front-proxy-client
```

#### kube-controller-manager 证书

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
   -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

#### kube-scheduler 证书

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
   -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

#### kube-proxy 证书

```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "system:kube-proxy",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
   -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   kube-proxy-csr.json | cfssljson -bare kube-proxy
```

#### kube-admin 证书

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "TS": "Beijing",
      "L": "Beijing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert \
   -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare admin
```

#### 生成 ServiceAccount Key

```bash
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub
```

#### 拷贝证书

```bash
mv *.pem ../
mv sa.pub ../
mv sa.key ../
```

## 部署容器运行时

**注意：docker 跟 containerd 选一种就行**

### docker

```bash
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
```

#### 设置国内镜像源

```bash
cat > /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com"
    ],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "3"
    },
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ],
    "storage-driver": "overlay2"
}
EOF
systemctl restart docker
```

#### 部署 cri-docker

##### 下载二进制包

```bash
mkdir -p /opt/bin/cri/dockerd
cd /opt/bin/cri/dockerd

wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.0/cri-dockerd-0.3.0.amd64.tgz

tar zxvf cri-dockerd-0.3.0.amd64.tgz

cd cri-dockerd-0.3.0.amd64/cri-dockerd
chmod +x cri-dockerd
cp cri-docker /opt/bin/cri/dockerd/
```

##### systemd 管理

```bash
cat > /lib/systemd/system/cri-docker.service <<EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/opt/bin/cri/dockerd/cri-dockerd --network-plugin=cni --pod-infra-container-image=imaxun/pause:3.9
ExecReload=/bin/kill -s HUP 
TimeoutSec=0
RestartSec=2
Restart=always

StartLimitBurst=3

StartLimitInterval=60s

LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF


cat > /lib/systemd/system/cri-docker.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF

systemctl daemon-reload ; systemctl enable cri-docker --now
```

### containerd

#### 下载二进制包

```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.15/cri-containerd-cni-1.6.15-linux-amd64
wget https://github.com/moby/buildkit/releases/download/v0.11.0/buildkit-v0.11.0.linux-amd64.tar.gz
wget https://github.com/containerd/nerdctl/releases/download/v1.1.0/nerdctl-1.1.0-linux-amd64.tar.gz

tar zxvf cri-containerd-cni-1.6.15-linux-amd64
cp cri-containerd-cni-1.6.15-linux-amd64/usr/local/bin/* /usr/local/bin/
cp cri-containerd-cni-1.6.15-linux-amd64/usr/local/sbin/* /usr/local/sbin/

tar zxvf buildkit-v0.11.0.linux-amd64.tar.gz
cp buildkit-v0.11.0.linux-amd64/bin/* /usr/local/bin/

tar zxvf nerdctl-1.1.0-linux-amd64.tar.gz
cp nerdctl-1.1.0-linux-amd64/nerdctl /usr/local/bin/

export PATH=$PATH:/usr/local/bin:/usr/local/sbin 
source /etc/profile
```

#### 配置文件

**containerd 配置文件**

```bash
mkdir -p /etc/containerd 
containerd config default > /etc/containerd/config.toml 

sandbox_image = "imaxun/pause:3.9"

SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

      
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
           endpoint = ["https://quay.tencentcloudcr.com"]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
           endpoint = ["https://registry-1.docker.io", "https://mirror.ccs.tencentyun.com"]
```

**buildkit 配置文件**

```bash
mkdir -p /etc/buildkit 
cat > /etc/buildkit/buildkitd.toml <<EOF
debug = true
# root is where all buildkit state is stored.
root = "/var/lib/buildkit"

[worker.oci]
  enabled = false
  
[worker.containerd]
  address = "/run/containerd/containerd.sock"
  enabled = true
  platforms = [ "linux/amd64"]
  namespace = "default"
  gc = true
  # gckeepstorage sets storage limit for default gc profile, in MB.
  gckeepstorage = 9000
  # maintain a pool of reusable CNI network namespaces to amortize the overhead
  # of allocating and releasing the namespaces
  cniPoolSize = 16
EOF
```

**nerdctl 配置文件**

```bash
mkdir -p /etc/nerdctl 
cat > /etc/nerdctl/nerdctl.toml <<EOF
# This is an example of /etc/nerdctl/nerdctl.toml .
# Unrelated to the daemon's /etc/containerd/config.toml .

debug          = false
debug_full     = false
address        = "unix:///run/containerd/containerd.sock"
namespace      = "k8s.io"
snapshotter    = "overlayfs"
cgroup_manager = "cgroupfs"
hosts_dir      = ["/etc/containerd/certs.d", "/etc/docker/certs.d"]
experimental   = true
cni_path       = "/opt/cni/bin" 
EOF  

export NERDCTL_TOML=/etc/nerdctl/nerdctl.toml
source /etc/profile
```

#### systemd 管理

**containerd 服务**

```bash
cat > /lib/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
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

systemctl daemon-reload ; systemctl enable containerd --now
```

**buildkitd 服务**

```bash
cat > /lib/systemd/system/buildkit.service <<EOF
[Unit]
Description=BuildKit
Requires=buildkit.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
Type=notify
ExecStart=/usr/local/bin/buildkitd --addr fd:// --config=/etc/buildkit/buildkitd.toml

[Install]
WantedBy=multi-user.target
EOF
cat > /lib/systemd/system/buildkit.socket <<EOF
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Socket]
ListenStream=%t/buildkit/buildkitd.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
EOF

systemctl daemon-reload ; systemctl enable buildkit --now
```

## 部署 Etcd 集群

**以下部署步骤在规划的三个 etcd 节点操作一样，唯一不同的是 etcd 配置文件中的服务器 IP 要写当前的：**

### 下载二进制包

```bash
mkdir -p /opt/bin/etcd
mkdir -p /opt/cfg/etcd
cd /opt/bin/etcd

wget https://github.com/etcd-io/etcd/releases/download/v3.5.6/etcd-v3.5.6-linux-amd64.tar.gz

tar zxvf etcd-v3.5.6-linux-amd64.tar.gz

chmod +x etcdctl etcd

vi /etc/profile
export PATH=$PATH:/opt/bin/etcd
source /etc/profile
```

### 配置文件

```bash
cat > /opt/cfg/etcd/etcd-conf.yml <<EOF
# 配置文档参考 https://doczhcn.gitbook.io/etcd/index/index-1/configuration 
name: 'etcd01'
data-dir: /opt/data/etcd
wal-dir: /opt/data/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://172.17.0.17:2380'
listen-client-urls: 'https://172.17.0.17:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://1.117.115.10:2380'
advertise-client-urls: 'https://1.117.115.10:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'etcd01=https://1.117.115.10:2380'
initial-cluster-token: 'etcd-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/opt/certificate/etcd/etcd-server.pem'
  key-file: '/opt/certificate/etcd/etcd-server-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/opt/certificate/etcd/ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/opt/certificate/etcd/etcd-server.pem'
  key-file: '/opt/certificate/etcd/etcd-server-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/opt/certificate/etcd/ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
EOF
```

### systemd 管理

```bash
cat > /usr/lib/systemd/system/etcd.service  <<\EOF
[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/opt/bin/etcd/etcd --config-file=/opt/cfg/etcd/etcd-conf.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now etcd.service
```

### 检查状态

```bash
ETCDCTL_API=3 /opt/bin/etcd/etcdctl  \
--cacert=/opt/certificate/etcd/ca.pem  \
--cert=/opt/certificate/etcd/etcd-client.pem \
--key=/opt/certificate/etcd/etcd-client-key.pem  \
--endpoints="https://1.117.115.10:2379" endpoint health --write-out=table
```

### 指定网段

```bash
ETCDCTL_API=3 /opt/bin/etcd/etcdctl  \
--cacert=/opt/certificate/etcd/ca.pem  \
--cert=/opt/certificate/etcd/etcd-client.pem \
--key=/opt/certificate/etcd/etcd-client-key.pem  \
--endpoints="https://1.117.115.10:2379" put /coreos.com/network/config \  
'{ "Network": "172.1.0.0/16", "Backend": {"Type": "vxlan"}}'
```

## 部署 kubernetes 组件

### 下载二进制包

```bash
mkdir -p /opt/bin/kubernetes/master
mkdir -p /opt/bin/kubernetes/node
mkdir -p /opt/cfg/kubernetes/master
mkdir -p /opt/cfg/kubernetes/node
mkdir -p /opt/cfg/kubernetes/admin
cd /opt/bin/kubernetes

wget https://dl.k8s.io/v1.26.0/kubernetes-server-linux-amd64.tar.gz

tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes-server-linux-amd64/kubernetes/server/bin/kube-apiserver

chmod +x kubectl kube-apiserver kube-controller-manager kube-scheduler kube-proxy kube-proxy

cp -p kubectl /opt/bin/kubernetes/
cp -p kube-apiserver kube-controller-manager kube-scheduler /opt/bin/kubernetes/master/
cp -p kube-proxy kube-proxy /opt/bin/kubernetes/node/

vi /etc/profile
export PATH=$PATH:/opt/bin/kubernetes
source /etc/profile

source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 部署 master 组件

#### kube-apiserver 组件

##### 生成 token

```bash
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > /opt/cfg/kubernetes/admin/token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

##### 配置文件

```bash
cat > /opt/cfg/kubernetes/master/kube-apiserver.conf <<EOF
# 参数说明 https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-apiserver/ 
KUBE_APISERVER_OPTS="--v=2  \
--allow-privileged=true  \
--bind-address=0.0.0.0  \
--secure-port=6443  \
--advertise-address=1.117.115.10 \
--service-cluster-ip-range=10.0.0.0/24  \
--service-node-port-range=1-50000  \
--etcd-servers=http://127.0.0.1:2379 \
--etcd-cafile=/opt/certificate/etcd/ca.pem  \
--etcd-certfile=/opt/certificate/etcd/etcd-client.pem  \
--etcd-keyfile=/opt/certificate/etcd/etcd-client-key.pem  \
--client-ca-file=/opt/certificate/kubernetes/ca.pem  \
--tls-cert-file=/opt/certificate/kubernetes/kube-apiserver.pem  \
--tls-private-key-file=/opt/certificate/kubernetes/kube-apiserver-key.pem  \
--kubelet-client-certificate=/opt/certificate/kubernetes/kube-apiserver.pem  \
--kubelet-client-key=/opt/certificate/kubernetes/kube-apiserver-key.pem  \
--service-account-key-file=/opt/certificate/kubernetes/sa.pub  \
--service-account-signing-key-file=/opt/certificate/kubernetes/sa.key  \
--service-account-issuer=https://kubernetes.default.svc.cluster.local \
--kubelet-preferred-address-types=Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
--authorization-mode=Node,RBAC  \
--enable-bootstrap-token-auth=true  \
--token-auth-file=/opt/cfg/kubernetes/admin/token.csv \
--requestheader-client-ca-file=/opt/certificate/kubernetes/front-proxy-ca.pem  \
--proxy-client-cert-file=/opt/certificate/kubernetes/front-proxy-client.pem  \
--proxy-client-key-file=/opt/certificate/kubernetes/front-proxy-client-key.pem  \
--requestheader-allowed-names=aggregator  \
--requestheader-group-headers=X-Remote-Group  \
--requestheader-extra-headers-prefix=X-Remote-Extra-  \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true"
EOF
```

##### systemd 管理

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
EnvironmentFile=/opt/cfg/kubernetes/master/kube-apiserver.conf
ExecStart=/opt/bin/kubernetes/master/kube-apiserver $KUBE_APISERVER_OPTS

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-apiserver.service
systemctl restart kube-apiserver.service
```

#### kube-controller-manager 组件

##### 配置文件

```bash
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/certificate/kubernetes/ca.pem \
     --embed-certs=true \
     --server=https://1.117.115.10:6443 \
     --kubeconfig=/opt/cfg/kubernetes/master/kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/opt/certificate/kubernetes/kube-controller-manager.pem \
     --client-key=/opt/certificate/kubernetes/kube-controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/opt/cfg/kubernetes/master/kube-controller-manager.kubeconfig

kubectl config set-context default  \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/opt/cfg/kubernetes/master/kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=/opt/cfg/kubernetes/master/kube-controller-manager.kubeconfig

cat > /opt/cfg/kubernetes/master/kube-controller-manager.conf <<EOF
# 参数说明 https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-controller-manager/
KUBE_CONTROLLER_MANAGER_OPTS="--v=2 \
--bind-address=127.0.0.1 \
--root-ca-file=/opt/certificate/kubernetes/ca.pem \
--cluster-signing-cert-file=/opt/certificate/kubernetes/ca.pem \
--cluster-signing-key-file=/opt/certificate/kubernetes/ca-key.pem \
--service-account-private-key-file=/opt/certificate/kubernetes/sa.key \
--kubeconfig=/opt/cfg/kubernetes/master/kube-controller-manager.kubeconfig \
--leader-elect=true \
--use-service-account-credentials=true \
--node-monitor-grace-period=40s \
--node-monitor-period=5s \
--pod-eviction-timeout=2m0s \
--controllers=*,bootstrapsigner,tokencleaner \
--allocate-node-cidrs=true \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-cidr=172.1.0.0/16 \
--node-cidr-mask-size-ipv4=24 \
--requestheader-client-ca-file=/opt/certificate/kubernetes/front-proxy-ca.pem"
EOF
```

##### systemd 管理

```bash
cat > /usr/lib/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
EnvironmentFile=/opt/cfg/kubernetes/master/kube-controller-manager.conf
ExecStart=/opt/bin/kubernetes/master/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl restart kube-controller-manager
```

#### kube-scheduler 组件

##### 配置文件

```bash
kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/certificate/kubernetes/ca.pem \
     --embed-certs=true \
     --server=https://1.117.115.10:6443 \
     --kubeconfig=/opt/cfg/kubernetes/master/kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/opt/certificate/kubernetes/kube-scheduler.pem \
     --client-key=/opt/certificate/kubernetes/kube-scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/opt/cfg/kubernetes/master/kube-scheduler.kubeconfig

kubectl config set-context default \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/opt/cfg/kubernetes/master/kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=/opt/cfg/kubernetes/master/kube-scheduler.kubeconfig

cat > /opt/cfg/kubernetes/master/kube-scheduler.conf <<EOF
KUBE_SCHEDULER_OPTS="--v=2 \
--leader-elect=true \
--kubeconfig=/opt/cfg/kubernetes/master/kube-scheduler.kubeconfig  \
--bind-address=127.0.0.1"
EOF
```

##### systemd 管理

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
EnvironmentFile=/opt/cfg/kubernetes/master/kube-scheduler.conf
ExecStart=/opt/bin/kubernetes/master/kube-scheduler $KUBE_SCHEDULER_OPTS

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-scheduler
systemctl restart kube-scheduler
```

#### 生成 admin.kubeconfig

```bash
mkdir /root/.kube/ -p


kubectl config set-cluster kubernetes     \
  --certificate-authority=/opt/certificate/kubernetes/ca.pem     \
  --embed-certs=true     \
  --server=https://1.117.115.10:6443     \
  --kubeconfig=/opt/cfg/kubernetes/admin/admin.kubeconfig

kubectl config set-credentials admin  \
  --client-certificate=/opt/certificate/kubernetes/admin.pem \
  --client-key=/opt/certificate/kubernetes/admin-key.pem \
  --embed-certs=true     \
  --kubeconfig=/opt/cfg/kubernetes/admin/admin.kubeconfig

kubectl config set-context default     \
  --cluster=kubernetes     \
  --user=admin \
  --kubeconfig=/opt/cfg/kubernetes/admin/admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig

cp /opt/cfg/kubernetes/admin/admin.kubeconfig  /root/.kube/config
```

#### 检查状态

```bash
kubectl get cs
NAME                 STATUS    MESSAGE                         ERROR
etcd-0               Healthy   {"health":"true","reason":""}   
controller-manager   Healthy   ok                              
scheduler            Healthy   ok 
```

### 部署 node 组件

#### kubelet 组件

##### 创建 TLS Bootstrapping 认证文件

```bash
a=`head -c 16 /dev/urandom | od -An -t x | tr -d ' ' | head -c6`
b=`head -c 16 /dev/urandom | od -An -t x | tr -d ' ' | head -c16`

cat > /opt/cfg/kubernetes/admin/bootstrap.secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-$a
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token generated by 'kubelet '."
  token-id: $a
  token-secret: $b
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups:  system:bootstrappers:default-node-token,system:bootstrappers:worker,system:bootstrappers:ingress
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-bootstrapper
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-certificate-rotation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF

kubectl config set-cluster kubernetes  \
--certificate-authority=../ca/ca.pem   \
--embed-certs=true   \
--server=https://127.0.0.1:6443   \
--kubeconfig=bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user  \
--token=$a.$b \
--kubeconfig=bootstrap-kubelet.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes \
--cluster=kubernetes   \
--user=tls-bootstrap-token-user  \
--kubeconfig=bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes  \
--kubeconfig=bootstrap-kubelet.kubeconfig

kubectl apply -f bootstrap.secret.yaml
```

##### 配置文件

```bash
cat > /opt/cfg/kubernetes/node/kubelet.conf <<EOF
KUBELET_OPTS="--v=2 \
--hostname-override=qcloud-host01 \
--bootstrap-kubeconfig=/opt/cfg/kubernetes/admin/bootstrap-kubelet.kubeconfig  \
--kubeconfig=/opt/cfg/kubernetes/node/kubelet.kubeconfig \
--config=/opt/cfg/kubernetes/node/kubelet-conf.yml \
--container-runtime-endpoint=unix:///run/cri-dockerd.sock \
--pod-infra-container-image=imaxun/pause:3.9  \
--cert-dir=/opt/cfg/kubernetes/node/certificate"
EOF

cat > /opt/cfg/kubernetes/node/kubelet-conf.yml <<EOF
# 参数说明 https://kubernetes.io/zh-cn/docs/reference/config-api/kubelet-config.v1beta1/
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/certificate/kubernetes/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /opt/cfg/kubernetes/node/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF
```

##### systemd 管理

```bash
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/cfg/kubernetes/node/kubelet.conf
ExecStart=/opt/bin/kubernetes/node/kubelet $KUBELET_OPTS

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
```

#### kube-proxy 组件

##### 配置文件

```bash
cat > /opt/cfg/kubernetes/node/kubelet.conf <<EOF
KUBE_PROXY_OPTS="--v=2  \
--hostname-override=qcloud-host01  \
--config=/opt/cfg/kubernetes/node/kube-proxy-conf.yml"
EOF

cat > /opt/cfg/kubernetes/node/kube-proxy-conf.yml <<EOF
#配置说明https://kubernetes.io/zh-cn/docs/reference/config-api/kube-proxy-config.v1alpha1/
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ''
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /opt/cfg/kubernetes/node/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.1.0.0/16
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: qcloud-host01
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 5s
  syncPeriod: 30s
ipvs:
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
EOF


kubectl config set-cluster kubernetes \
     --certificate-authority=/opt/certificate/kubernetes/ca.pem \
     --embed-certs=true \
     --server=https://1.117.115.10:6443 \
     --kubeconfig=/opt/cfg/kubernetes/node/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
     --client-certificate=/opt/certificate/kubernetes/kube-proxy.pem \
     --client-key=/opt/certificate/kubernetes/kube-proxy-key.pem \
     --embed-certs=true \
     --kubeconfig=/opt/cfg/kubernetes/node/kube-proxy.kubeconfig

kubectl config set-context default \
     --cluster=kubernetes \
     --user=system:kube-proxy \
     --kubeconfig=/opt/cfg/kubernetes/node/kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=/opt/cfg/kubernetes/node/kube-proxy.kubeconfig
```

##### systemd 管理

```bash
cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/cfg/kubernetes/node/kubelet.conf
ExecStart=/opt/bin/kubernetes/node/kubelet $KUBELET_OPTS

[Install]
WantedBy=multi-user.target[root@qcloud-host01 node]# cat /usr/lib/systemd/system/kube-proxy.service 
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
EnvironmentFile=/opt/cfg/kubernetes/node/kube-proxy.conf
ExecStart=/opt/bin/kubernetes/node/kube-proxy $KUBE_PROXY_OPTS

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
```

#### 检查状态

```bash
kubectl get node
NAME            STATUS   	ROLES            AGE     VERSION
qcloud-host01   NotReady    none   			     	 v1.26.0
```

## 部署网络组件

```bash
mkdir -p /opt/cni/bin
cd /opt/cni/bin

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

tar zxvf cni-plugins-linux-amd64-v1.1.1.tgz

cd cni-plugins-linux-amd64-v1.1.1
chmod +x *
cp * /opt/cni/bin/
wget https://github.com/flannel-io/flannel/blob/v0.20.2/Documentation/kube-flannel.yml

  net-conf.json: |
    {
      "Network": "172.1.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }    

kubectl apply -f kube-flannel.yml

kubectl get node
NAME            STATUS   	ROLES            AGE     VERSION
qcloud-host01   Ready    	none   			     	 v1.26.0

kubectl annotate node qcloud-host01 flannel.alpha.coreos.com/public-ip-overwrite=1.117.115.10 --overwrite
```

## 部署 coredns 组件

```bash
wget https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed

  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
        log
    }
    
clusterIP: 10.0.0.2  

kubectl apply -f coredns.yaml
```

## 部署 ingress-nginx

```bash
wget https://github.com/kubernetes/ingress-nginx/blob/controller-v1.5.1/deploy/static/provider/baremetal/deploy.yaml

registry.k8s.io/ingress-nginx/controller:v1.5.1 替换为 chenmo/controller:v1.5.1 
registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343 替换为 chenmo/kube-webhook-certgen:v20220916-gd32f8c343


apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller 
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx

kubectl apply -f deploy.yaml
```

## 其他命令

```bash
kubectl label nodes qcloud-host01 linux=qcloud-host01
kubectl label nodes qcloud-host01 node-role.kubernetes.io/lb=lb-qcloud-host01
kubectl label nodes qcloud-host01 node-role.kubernetes.io/master=
kubectl label nodes qcloud-host01 node-role.kubernetes.io/node=qcloud-host01
kubectl label nodes qcloud-node01 linux=qcloud-node01
kubectl label nodes qcloud-node01 node-role.kubernetes.io/node=qcloud-node01

kubectl create secret docker-registry qcloud --docker-server=ccr.ccs.tencentyun.com --docker-username=xxx --docker-password=xxx -n default
```

> [!TOP]
> 可以使用 [[Attachments/calico.yaml]]