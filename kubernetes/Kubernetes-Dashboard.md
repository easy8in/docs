[官方仓库](http://github.com/kubernetes/dashboard)

```bash
# master 节点
## 获取配置
curl -o kube-dashboard.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

## 修改配置
### spec type: NodePort
###   nodePort: 30001
### 内容如下： 
vim kube-dashboard.yaml
#>>> 
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
#<<<

## 部署
kubectl apply -f kube-dashboard.yaml

## 获取tokken
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep kubernetes-dashboard | awk '{print $1}')
## 命令说明 $() 是shell命令 {}内是变量名
## kubectl -n {namespace} describe secret $(kubectl -n {namespace} get secret | grep {usename} | awk '{print $1}')

## 访问地址
https://${master-ip}:30001
## 输入token值，登录系统
```