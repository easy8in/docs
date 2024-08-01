## 准备工作

```bash
# Master 开放防火墙端口
ufw allow 6443/tcp
ufw allow 2379:2380/tcp
ufw allow 10250/tcp
ufw allow 10259/tcp
ufw allow 10257/tcp

# 各节点
ufw allow 10250/tcp
ufw allow 30000:32767/tcp

# 安装kubernetes 基础工具
sudo apt get kubeadm kubelet kubectl

# 生成初始化配置文件
# kubeadm config print init-defaults > kubeadm.yaml

# 初始化集群
kubeadm init \
--kubernetes-version=v1.26.3 \
--control-plane-endpoint=192.168.18.201 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--cri-socket unix:///var/run/cri-dockerd.sock \
| tee kubeadm-init.log

# root 用户安装
vim /etc/profile
export KUBECONFIG=/etc/kubernetes/admin.conf
source /etc/profile

# 子节点加入集群
kubeadm join 192.168.85.40:6443 --token g39rrs.opriptbt3z8j1sar \
	--discovery-token-ca-cert-hash sha256:7d8ae94c2ad5d651b7db45f9820e0bc94de09d50636e308abb9d7af0552de9b0

# 配置 flannel网络
curl -o kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl apply -f kube-flannel.yml
```