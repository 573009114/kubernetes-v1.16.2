

[部署etcd](https://github.com/573009114/Kubernetes.install/blob/master/No.03%20%E5%BF%AB%E9%80%9F%E9%83%A8%E7%BD%B2etcd%E6%9C%8D%E5%8A%A1%EF%BC%88%E5%B8%A6%E8%AF%81%E4%B9%A6%EF%BC%89.md)

[安装指定版本docker](https://github.com/573009114/Kubernetes.install/blob/master/No.05%20%E5%AE%89%E8%A3%85%E6%8C%87%E5%AE%9A%E7%89%88%E6%9C%ACdocker.md)
#### 准备3台主节点：km1/km2/km3

 ```
# 添加镜像源
vim /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0

 
yum install socat kubelet kubeadm kubectl kubernetes-cni -y

systemctl enable kubelet && systemctl start kubelet

#下载镜像组件
docker pull gotok8s/kube-apiserver:v1.16.2
docker pull gotok8s/kube-controller-manager:v1.16.2
docker pull gotok8s/kube-scheduler:v1.16.2
docker pull gotok8s/kube-proxy:v1.16.2

#修改镜像名
docker tag 8454cbe08dc9 k8s.gcr.io/kube-proxy:v1.16.2
docker tag ebac1ae204a2 k8s.gcr.io/kube-scheduler:v1.16.2
docker tag c2c9a0406787 k8s.gcr.io/kube-apiserver:v1.16.2
docker tag 6e4bffa46d70 k8s.gcr.io/kube-controller-manager:v1.16.2

#删除多余镜像
docker rmi gotok8s/kube-proxy:v1.16.2
docker rmi gotok8s/kube-apiserver:v1.16.2
docker rmi gotok8s/kube-controller-manager:v1.16.2
docker rmi gotok8s/kube-scheduler:v1.16.2

 ```

1.编辑kubeadm-config.yaml
```
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.2
controlPlaneEndpoint: kube.cluster:6443  # haproxy地址及端口
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers # 指定镜像源为阿里源
apiServer:
  certSANs:
  - 10.20.55.168
  - 127.0.0.1
  - k8s.master
networking:
  podSubnet: 10.68.0.0/16
  serviceSubnet: 10.244.0.0/16
etcd:
    external:
        endpoints:
        - https://10.20.55.168:2379
        - https://10.20.55.169:2379
        - https://10.20.55.170:2379
        caFile: /etc/kubernetes/pki/etcd-ca.pem
        certFile: /etc/kubernetes/pki/etcd.pem
        keyFile: /etc/kubernetes/pki/etcd-key.pem
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
ipvs:
  minSyncPeriod: 1s
  scheduler: rr
  syncPeriod: 10s
mode: ipvs
```

2. 编辑/etc/host
```
10.10.0.21  kube.cluster km1
10.10.0.22  kube.cluster km2
10.10.0.23  kube.cluster km3
```

3.初始化机器
```
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --experimental-upload-certs
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
4.加入节点
```
kubeadm join kube.cluster:6443 --token h599ue.pb1vil5hknveorqg \
    --discovery-token-ca-cert-hash sha256:90732df9c15c7eeb4b3e25f5767b3014e449add94438c532434d1865d2c4f79b
```
