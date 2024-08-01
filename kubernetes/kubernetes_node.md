### kubernetes Node 节点加入集群

```bash
# 开放防火墙端口(产品环境)
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
firewall-cmd --reload

# 安装必要组件
yum install -y kubeadm kubelet
systemctl enable kubelet

## master 节点打印token
kubeadm token list
kubeadm token create

# node join
kubeadm join 192.168.85.40:6443 --token g39rrs.opriptbt3z8j1sar \
	--discovery-token-ca-cert-hash sha256:7d8ae94c2ad5d651b7db45f9820e0bc94de09d50636e308abb9d7af0552de9b0
```