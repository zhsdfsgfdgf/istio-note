@[TOC]
## 一.准备环境 
Istio1.5支持的kubernetes版本

 - 1.14-1.16
 
kubernetes环境
 - 本地(Minikube,VM)
 - 云平台(AWS)
 - 在线playground（katacoda）、https://www.katacoda.com/courses/kubernetes/playground

 ## 二.下载安装Istio
 
 **下载istio包**
 
到Istio的发布页面https://github.com/istio/istio/releases ，查找当前可用版本的对应操作系统的包

```
wget https://github.com/istio/istio/releases/download/1.5.2/istio-1.5.2-linux.tar.gz
```

**安装目录包含如下内容**

```
install/kubernetes 目录下，有 Kubernetes 相关的 YAML 安装文件
samples/ 目录下，有示例应用程序
bin/ 目录下，包含 istioctl 的客户端文件
```


**将istioctl加入环境变量**

```
进入下载目录
export PATH=$PWD/bin:$PATH
验证
istioctl version
```
**安装demo配置**

```
 istioctl manifest apply --set profile=demo
 查看pod
 kubectl get pod -n istio-system
 确保关联的 Kubernetes pod 已经部署，并且 STATUS 为 Running
```

## 三.自动注入
安装 Istio 后，就可以部署您自己的服务，或部署安装程序中系统的任意一个示例应用
当使用 kubectl apply 来部署应用时，如果 pod 启动在标有 istio-injection=enabled 的命名空间中，那么，Istio sidecar 注入器将自动注入 Envoy 容器到应用的 pod 中

```
开启default命名空间的Istio自动注入功能
kubectl label namespace default istio-injection=enabled
```
## 四.部署官方bookinfo示例

```
 kubectl apply -f  ./samples/bookinfo/platform/kube/bookinfo.yaml
 
 查看部署状态
 kubectl get pod
 
 使用Gateway创建访问入口
 kubectl apply -f ./samples/bookinfo/networking/bookinfo-gateway.yaml
 
 查看Gateway
 kubectl get gateway
 
 获取访问地址。使用如下的命令获取访问入口地址
 export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
 export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
 export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')
 export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
 echo http://$GATEWAY_URL/productpage

使用浏览器访问
```
