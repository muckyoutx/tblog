title: docker&k8s的部署笔记
img: false
cover:  cover.jpg
top: false
abbrlink: 49176
date: 2021-05-22 20:43:33
summary:
tags: docker
          k8s
categories:  k8s文档
password:
---


1.升级内核   

```bash
 vi /etc/hostname         
    yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
    yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    yum --enablerepo=elrepo-kernel install kernel-lt
   cat /boot/grub2/grub.cfg
     vi /etc/default/grub
GRUB_DEFAULT=0     
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

 

2共通部分

```bash
  vi /etc/sysconfig/network-scripts/ifcfg-eno33554960    #nat外网
   vi /etc/sysconfig/network-scripts/ifcfg-eno16777736      #内网
systemctl restart network    #修改NIC后重启网络服务
  setenforce 0
   systemctl stop firewalld
    systemctl disable firewalld
     vi /etc/selinux/config= sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
修改SELINUX=disabled
vi /etc/hosts  #修改hosts映射
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.2.10 master
192.168.2.20 node1
192.168.2.30 node2
199.232.68.133 raw.githubusercontent.com
   scp hosts root@192.168.2.20:/etc/
    scp hosts root@192.168.2.30:/etc/      #远程拷贝hosts
swapoff –a
sed -i '/ swap / s/^/#/' /etc/fstab   #关闭交换分区

[root@master /] cat  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1


```

 可以scp

```bash
modprobe br_netfilter   #需执行此命令以便配置生效
sysctl -p /etc/sysctl.d/k8s.conf   #应用
yum install wget –y     # wget 安装
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo更新镜像
yum makecache   缓存
yum install -y yum-utils device-mapper-persistent-data lvm2  #安装工具
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  添加docker源
yum list docker-ce --showduplicates|sort 查看可安装版本
yum -y install docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io  #安装docker
cat << EOF > /etc/docker/daemon.json
    {
     "exec-opts": ["native.cgroupdriver=systemd"]
      } 
EOF    更改cgroupdriver为systemd

  scp  /etc/docker/daemon.json root@node1:/etc/docker/
    scp  /etc/docker/daemon.json root@node2:/etc/docker/

systemctl start docker
 systemctl enable docker
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
   name=Kubernetes
  baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
    enabled=1
   gpgcheck=0
   repo_gpgcheck=0
    gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
           http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF       #获取k8s源
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet.service

```

3 master部分

```bash
kubeadm init --kubernetes-version=v1.20.5 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap   #master初始化

mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
systemctl start kubelet  # master操作
```

 

4各个node部分

```bash
kubeadm join 192.168.2.10:6443 --token vlqx19.4l8z2xbll6fsv0pj     --discovery-token-ca-cert-hash sha256:58e2ac72e7d6a884d7ff5c6ab9b6fc763c903f20b09f3dbd769f2e30821ecaba              #    写入master生成的token  node操作
```

3继续master

```bash
kubectl get node 查看node信息发现notready 因为网络插件未安装
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  kubectl apply -f kube-flannel.yml    #安装flannel网络插件
```



（https://github.com/kubernetes/dashboard/  、 
https://github.com/kubernetes/dashboard/tree/master/aio/deploy下载或复制recommended.yaml）

```bash
kubectl apply -f recommended.yaml
kubectl delete service kubernetes-dashboard --namespace=kubernetes-dashboard  #删掉service

[root@master docker]# cat dashboard-svc.yaml    #重建dashboard-service
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
     nodePort: 30443
     targetPort: 8443
       selector:
         k8s-app: kubernetes-dashboard


kubectl apply -f dashboard-svc.yaml  #安装通过nodeport暴露的服务

[root@master docker]# cat dashboard-svc-account.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin

  namespace: kube-system
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
subjects:

  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io    #创建角色

kubectl apply –f dashboard-svc-account.yaml

kubectl get secret -n kube-system |grep admin
  kubectl describe secret dashboard-admin-token-qnzhh -n kube-system              #token查询
kubectl create clusterrolebinding test:anonymous --clusterrole=cluster-admin --user=system:anonymous    #允许查看命名空间
```




```bash
mkdir -p /root/kubernetes/Helm   && cd  /root/kubernetes/Helm  
wget https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz
tar -zxvf helm-v3.5.3-linux-amd64.tar.gz 
   mv linux-amd64/helm /usr/local/bin/helm
    helm  version
   helm repo add stable https://charts.helm.sh/stable

wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/baremetal/deploy.yaml
docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.33.0   #拉取ingress-controller镜像
vi deploy.yaml  #将controller的image地址改为quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.33.0
kubectl apply –f deploy.yaml
[root@master docker]# cat myapp.yaml 
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  ports:

  - name: http
    port: 80
    targetPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata: 
  name: myapp-deploy
spec:
  replicas: 2
  selector: 
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v2
        ports:
        - name: httpd
          containerPort: 80


[root@master docker]# cat mysvc.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-myapp
  namespace: default
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:

  - host: myapp.muckyou.com #生产中该域名应当可以被公网解析
    http:
      paths:
      - path: 
        backend:
          serviceName: myapp
```


几个基础运维命令：

```bash
kubectl get node
   kubectl get pods --all-namespaces
   kubectl get pods --all-namespaces -o wide
kubectl get svc --all-namespaces
   kubectl get svc--all-namespaces -o wide
kubectl get deployment --all-namespaces
   kubectl get deployment --all-namespaces -o wide
kubectl get ingress --all-namespaces


kubectl  exec -ti  myapp-deploy-9c59596cd-45xsv -n default  -- /bin/sh

kubectl delete {pod|svc|deployment|ingress} myapp-deploy-7b85dc6654-wlb8v
```

