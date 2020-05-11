## 网关

可以把他认为是一个运行在网格边缘的负载均衡器，接收外部请求，转发给网格内的服务，另外，我们还会通过网关来配置对外的端口，协议与内部服务的映射关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020051120550824.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTA5OTc4,size_16,color_FFFFFF,t_70)
## istio中的两种网关

Ingress网关，控制进入流量

Egress网关，控制出口流量

## 配置项

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway 
  servers: //定义入口点,包括port,hosts,tls,defaultEndpoint
  - port: //端口,协议,以及名称
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

```

## 任务
说明：创建一个入口网关，将进入网格的流量分到不同地址
目标：学会用gateway控制入口流量，掌握gateway的配置方法

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: test-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: test-gateway
spec:
  hosts:
  - "*"
  gateways:
  - test-gateway #注意与网关对应
  http:
  #有两个可以被暴露出来的url
  #虚拟服务会把匹配到这两个url的请求,打到具体的detail服务中
  - match:
    - uri:
        prefix: /details
    - uri:
        exact: /health
    route:
    - destination:
        host: details
        port:
          number: 9080
```
## 访问

```yaml
http://192.168.26.31:31245/details/0
http://192.168.26.31:31245/health
```
## 应用场景

 - 暴露网格内服务给外界
 - 配置访问安全相关的设置(HTTPS,mTLS等)
 - 作为应用的统一入口或者api聚合