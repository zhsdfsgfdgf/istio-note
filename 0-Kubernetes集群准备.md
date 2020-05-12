## 安装Kubeadm
Kubeadm用于创建Kubernetes集群

1）配置阿里云Kubernetes镜像源

```bash
/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```
2）安装Kubeadm相关软件

```bash
yum install kubelet-1.15.3 kubeadm-1.15.3 kubectl-1.15.3 --disableexcludes=kubernetes  -y
最后的选项意思是排除yum源里和kubernentes里相冲突的一些包
```
## 配置基础环境
1）关闭防火墙和SELinux

验证getenforce Disabled
```bash
$ systemctl stop firewalld
$ systemctl disable firewalld
$ setenforce 0
```
2）关闭Swap

```bash
$ swapoff /dev/xxx
并在/etc/fstab中注释掉swap（swapon -s查看并关闭）
```
3）配置Hosts解析

```bash
三台机器的/etc/hosts文件
192.168.26.30 vms30.rhce.cc vms30
192.168.26.31 vms31.rhce.cc vms31
192.168.26.32 vms32.rhce.cc vms32
```
4）RHEL/CentOS 7需要的特殊配置
开启一些转发功能，几乎所有的数据包通信都是通过iptables做转发的(三个节点都需要)安装k8s的时候，会自动创建iptables规则

```bash
/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1

```
5）配置Kubelet

```bash
systemctl restart kubelet ; systemctl enable kubelet
```

 - firewall-cmd --set-default-zone=trusted允许所有数据包是能够直接通信的firewall-cmd--list-all查看，因为安装k8s的时候，会自动创建防火墙规则，装好之后，不能iptables -F清除防火墙规则
 - 在安装kubernetes时候，一定要关闭swap，否则安装不下去，即使安装好了，也不能启动
 - 除了kublet外，其他都是已Pod（容器）的方式运行，所以每个节点都要开机启动kubelet
## 开始部署
<1>阿里云后台开放
 6443 2379 2380 10250~10252, 31620 端口

<2>在maser上执行 

```bash
kubeadm init  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version=v1.17.2（v1.15.3） --pod-network-cidr=10.244.0.0/16
```
不指定仓库，会发现下载的很慢
<3>按照提示执行命令：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
<4>加入节点

```bash
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.26.30:6443 --token 6slsrv.w8xb4uvc87he80yn \
    --discovery-token-ca-cert-hash sha256:a711b44489854ef7bc16e54ae4bd48257bec5db7f6a68ccea01091af353608be
```
<5>查看节点

```bash
[root@vms30 ~]# kubectl get nodes
NAME            STATUS     ROLES    AGE   VERSION
vms30.rhce.cc   NotReady   master   39m   v1.15.3
```
<6>部署网络
需要什么网络，去官网上下载提供的yaml文件就可以

```bash
kubectl apply -f rbac-kdd.yaml
kubectl apply -f calico.yaml
```
<7>查看集群

```bash
[root@vms30 ~]# kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
vms30.rhce.cc   Ready    master   5m40s   v1.15.3
vms31.rhce.cc   Ready    <none>   4m27s   v1.15.3
```
