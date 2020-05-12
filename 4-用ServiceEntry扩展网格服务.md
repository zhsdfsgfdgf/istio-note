## 服务入口
网关是将内部服务暴露给外部访问，服务入口正好相反，是把外部服务纳入到网格内部进行管理，主要是希望能够管理到外部服务的请求，比如需要对访问外部服务的请求做一些流量控制，还有就是能够帮我们扩展我们的网格，例如我们要给多个集群共享同一个网格(mesh)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200511215532395.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTA5OTc4,size_16,color_FFFFFF,t_70)
从图上可以看出，服务入口相当于抽象了一个外部服务，然后内部服务就像访问网格内部的服务一样去访问外部服务


## 任务
说明：将httpbin注册为网格内部的服务，并配置流控策略
目标：学会通过服务入口扩展网格，掌握服务入口的配置方法

## 配置
因为bookinfo服务里，没有带curl这个命令，因此想要模拟内部服务去请求外部服务，需要在添加另外一个服务，官方提供了sleep这样的服务
httpbin是一个非常精简的测试http请求的服务
如http://httpbin.org/headers可以看到头信息

```yaml
kubectl apply -f ./samples/sleep/sleep.yaml
kubectl exec -it sleep-f8cbf5b76-8fz8g -c sleep curl  http://httpbin.org
```



**关闭出流量访问权限**
outboundTrafficPolicy=REGISTER_ONLY
本来是ALLOW_ANY
istio中默认所有的网格内的服务，是允许直接访问外部服务的，所以我们先关闭到允许访问外部服务的方式，设置成只有注册过的服务才能访问外部服务

```yaml
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTER_ONLY/g' |kubectl replace -n istio-system -f -
```
这样的话发现在服务内部无法访问外部
然后定义一个服务入口，让sleep服务可以通过服务入口访问外部服务

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts: 外部服务的服务域名
  - httpbin.org
  ports: 定义具体的访问这个服务的协议及端口
  - number: 80
    name: http
    protocol: HTTP
  location: MESH_EXTERNAL 定义网格外部还是内部
  resolution: DNS 服务发现的机制,通过dns发现

```

```bash
[root@vms30 ~] kubectl get se
NAME          HOSTS           LOCATION        RESOLUTION   AGE
httpbin-ext   [httpbin.org]   MESH_EXTERNAL   DNS          27s

```
发现又可以访问了