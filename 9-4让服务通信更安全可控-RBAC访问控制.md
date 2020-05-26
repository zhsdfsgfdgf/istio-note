## 简介
Istio的授权功能使用的是基于角色的访问权限控制，在服务网格中提供了命名空间级别、服务级别以及方法级别的访问控制功能，具有如下特点：

 - 基于角色的语法语义，简单易用
 - 支持服务到服务和终端用户到服务的授权功能
 - 可以自定义属性，这样更灵活，例如在角色和角色绑定上使用条件
 - 高性能，Istio授权是在Envoy代理本地实施的。
授权架构如图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526114616398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzMTA5OTc4,size_16,color_FFFFFF,t_70)
管理员通过Istio的配置创建服务访问策略，Pilot接收策略规则后，生成Envoy代理需要的配置格式并分发给对应的Envoy代理，Envoy代理根据访问策略在本地执行服务访问权限的检查，判断服务的请求能否被通过
在Istio中，RBAC访问控制使用Kubernetes中的serviceaccount作为服务的身份标识，用于分配servicerole对象，因此在Kubernetes中部署服务时需要创建对应的serviceaccount，并给部署的deployment指定创建的serviceaccount。

## 启用授权
通过创建RbacConfig对象，可以启用RBAC授权功能。RbacConfig对象是整个网络中唯一的，且必须命名为default。在RbacConfig对象中可以通过指定mode参数来指定RBAC访问控制启用的范围，mode可取值如下：
 - OFF：完全关闭RBAC授权功能
 - ON：为所有的服务和命名空间启用RBAC授权功能
 - ON_WITH_INCLUSION：在指定命名空间或者服务上启用RBAC授权功能
 - ON_WITH_EXCLUSION：在指定命名空间或者服务之外的命名空间或者服务中启用RBAC授权功能。

在default命名空间启用RBAC授权功能

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: RbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: 
    - default
```
在istio-system和kube-system之外的命名空间中启用RBAC授权功能

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: RbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_EXCLUSION'
  exclusion:
    namespaces: 
    - istio-system
    - kube-system
```

## 授权策略
RBAC授权策略由ServiceRole和ServiceRoleBinding组成，ServiceRole中包含了服务访问权限的集合，ServiceRoleBinding把ServiceRole分配给指定的对象，如用户和服务。

ServiceRole对象中包含了字段为rules的权限列表，每一个rule有如下字段：

 - services：服务列表，可以设置为*表示指定命名空间的所有服务
 - methods：HTTP的方法列表，在gRPC请求中该字段会被忽略，因为gRPC中只有POST方法，可以设置为*表示不限定方法
 - paths：HTTP的请求URI列表或者gRPC请求中的方法列表，gRPC方法必须是/packageName.serviceName/methodName这种格式，并且是大小写敏感的。URI支持前缀、后缀和精确匹配，例如：/books/review精确匹配/books/review，前缀匹配/books/，后缀匹配/review。如果没有指定此字段表示对任何URI或者gRPC请求中的方法都生效
 - constraints：表示特殊限制列表，比如限制服务版本、请求头等信息

定义命名空间级别的ServiceRole示例如下：

```yaml
 1 apiVersion: rbac.istio.io/v1alpha1
 2 kind: ServiceRole
 3 metadata:
 4   name: service-viewer
 5   namespace: default
 6 spec:
 7   rules:
 8   - services: ["*"]
 9     methods: ["GET"]
10     constraints:
11     - key: "destination.labels[app]"
12       values:
13       - service-js
14       - service-python
15       - service-lua
16       - service-node
17       - service-go

```
如上的ServiceRole定义表示default命名空间的所有服务都有GET方法的权限，但是第10~17行又定义了特殊的限制，限制请求的目标地址只能为values中指定的服务，访问其他服务时没有权限。
ServiceRoleBinding的定义包含如下两个字段：

 - roleRef指定某个同命名空间的ServiceRole名字
 - subjects指定角色分配的主体对象列表。主体可以是user，也可以是properties，还可以是两者的组合。user的值一般是用户的标识或者服务的身份标识，也就是Pod使用的serviceaccount名称，使用如"cluster.local/ns/default/sa/service-python"这样的形式。properties与ServiceRole中的constraints字段的使用方法类似 
 
 定义命名空间级别的ServiceRoleBinding示例如下

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: bind-service-viewer
  namespace: default
spec:
  subjects:
  - properties:
      source.namespace: "istio-system"
  - properties:
      source.namespace: "default"
  roleRef:
    kind: ServiceRole
    name: "service-viewer"

```
如上代码中的ServiceRoleBinding定义，将名为service-viewer的ServiceRole分配给请求的来源命名空间为istio-system和default的请求主体

分配角色给所有用户示例：

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: bind-service-js-service-python-viewer
  namespace: default
spec:
  subjects:
  - user: "*"
  roleRef:
    kind: ServiceRole
    name: "service-js-service-python-viewer"
```
分配角色给所有认证的用户示例：

```yaml
apiVersion: rbac.istio.io/v1alpha1
kind: ServiceRoleBinding
metadata:
  name: bind-service-js-service-python-viewer
  namespace: default
spec:
  subjects:
  - properties:
      source.principal: "*"
  roleRef:
    kind: ServiceRole
    name: "service-js-service-python-viewer"
```
基于服务级别的访问控制策略使用示例：

```yaml

 1 apiVersion: rbac.istio.io/v1alpha1
 2 kind: ServiceRole
 3 metadata:
 4   name: service-js-service-python-viewer
 5   namespace: default
 6 spec:
 7   rules:
 8   - services:
 9     - "service-js.default.svc.cluster.local"
10     - "service-python.default.svc.cluster.local"
11     methods: ["GET"]
12 ---
13 apiVersion: rbac.istio.io/v1alpha1
14 kind: ServiceRoleBinding
15 metadata:
16   name: bind-service-js-service-python-viewer
17   namespace: default
18 spec:
19   subjects:
20   - user: "*"
21   roleRef:
22     kind: ServiceRole
23     name: "service-js-service-python-viewer"
24 ---
25 apiVersion: rbac.istio.io/v1alpha1
26 kind: ServiceRole
27 metadata:
28   name: service-lua-service-node-viewer
29   namespace: default
30 spec:
31   rules:
32   - services:
33     - "service-lua.default.svc.cluster.local"
34     - "service-node.default.svc.cluster.local"
35     methods: ["GET"]
36 ---
37 apiVersion: rbac.istio.io/v1alpha1
38 kind: ServiceRoleBinding
39 metadata:
40   name: bind-service-lua-service-node-viewer
41   namespace: default
42 spec:
43   subjects:
44   - user: "cluster.local/ns/default/sa/service-python"
45   roleRef:
46     kind: ServiceRole
47     name: "service-lua-service-node-viewer"
48 ---
49 apiVersion: rbac.istio.io/v1alpha1
50 kind: ServiceRole
51 metadata:
52   name: service-go-viewer
53   namespace: default
54 spec:
55   rules:
56   - services: ["service-go.default.svc.cluster.local"]
57     methods: ["GET"]
58 ---
59 apiVersion: rbac.istio.io/v1alpha1
60 kind: ServiceRoleBinding
61 metadata:
62   name: bind-service-go-viewer
63   namespace: default
64 spec:
65   subjects:
66   - user: "cluster.local/ns/default/sa/service-node"
67   roleRef:
68     kind: ServiceRole
69     name: "service-go-viewer"
```
第1~23行定义的访问控制策略表明，服务service-js和服务service-python可以被任何用户以GET方法访问。

第25~47行定义的访问控制策略表明，服务service-node和服务service-lua只可以被default命名空间中使用了名为service-python的serviceaccount的服务实例以GET方法访问。

第49~69行定义的访问控制策略表明，服务service-go只可以被default命名空间中使用了名为service-node的serviceaccount的服务实例以GET方法访问。

## 【实验】

1）部署服务：

```powershell
kubectl apply -f service/js/service-go.yaml
kubectl apply -f service/js/service-node.yaml
kubectl apply -f service/lua/service-lua.yaml
kubectl apply -f service/python/service-python.yaml
kubectl apply -f service/js/service-js.yaml
```

2）启用default命名空间的mTLS功能。

RBAC的部分实验需要服务身份认证，需要开启mTLS：

```powershell
kubectl apply -f istio/security/mtls-namespace-default-on.yaml
```

3）创建Gateway和service-js路由规则。

创建Gateway和服务service-js的mTLS路由规则，并路由到v1版本：

```powershell
kubectl apply -f istio/route/gateway-js-v1-mtls.yaml
```

4）替换service-node服务和service-python服务的部署为带有serviceaccount的Deployment：

```powershell
kubectl apply -f service/node/service-node-with-serviceaccount.yaml
kubectl apply -f service/python/service-python-with-serviceaccount.yaml
```
5）创建测试Pod

```powershell
kubectl apply -f kubernetes/dns-test.yaml
```
6）访问服务：

```powershell
kubectl exec dns-test -c dns-test -- curl -s http://service-go/env
{"message":"go v2"}
kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
{"message":"node v1","upstream":[{"message":"go v2","response_time":"0.40"}]}
kubectl exec dns-test -c dns-test -- curl -s http://service-lua/env
{"message":"lua v2"}
kubectl exec dns-test -c dns-test -- curl -s http://service-python/env
{"message":"python v1","upstream":[{"message":"lua v1","response_time":0.22},{"message":"node v1","response_time":0.47,"upstream":[{"message":"go v1","response_time":"0.24"}]}]}
```

此时服务开启了mTLS，所有服务访问正常。

7）浏览器访问。

浏览器访问地址为http://192.168.26.31:31148/ ，服务正常

8）启动default命名空间的RBAC访问控制：

```powershell
kubectl apply -f istio/security/rbac-config-on.yaml
```

9）访问服务：

```powershell
$ kubectl exec dns-test -c dns-test -- curl -s http://service-go/env
RBAC: access denied
$ kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
RBAC: access denied
$ kubectl exec dns-test -c dns-test -- curl -s http://service-lua/env
RBAC: access denied
$ kubectl exec dns-test -c dns-test -- curl -s http://service-python/env
RBAC: access denied
$ kubectl exec dns-test -c dns-test -- curl -s http://service-js/env
RBAC: access denied
```
此时由于开启了RBAC访问控制，而没有创建授权策略，所有服务访问都会被拒绝。

10）创建RBAC命名空间级别规则：

```powershell
kubectl apply -f istio/security/rbac-namespace-policy.yaml
```

11）访问服务：

```powershell
kubectl exec dns-test -c dns-test -- curl -s http://service-go/env
{"message":"go v1"}
kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
{"message":"node v2","upstream":[{"message":"go v1","response_time":"0.02"}]}
kubectl exec dns-test -c dns-test -- curl -s http://service-lua/env
{"message":"lua v1"}
kubectl exec dns-test -c dns-test -- curl -s http://service-python/env
{"message":"python v1","upstream":[{"message":"lua v2","response_time":0.06},{"message":"node v2","response_time":0.13,"upstream":[{"message":"go v1","response_time":"0.02"}]}]}
```

此时已经创建了授权策略，所有服务访问正常。

12）浏览器访问。

浏览器访问，服务正常

13）删除命名空间级别的授权：

```powershell
kubectl delete -f istio/security/rbac-namespace-policy.yaml
```

14）创建服务级别的授权：

```powershell
kubectl apply -f istio/security/rbac-service-policy.yaml
```

15）访问服务：

```powershell
kubectl exec dns-test -c dns-test -- curl -s http://service-go/env
RBAC: access denied
kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
RBAC: access denied
kubectl exec dns-test -c dns-test -- curl -s http://service-lua/env
RBAC: access denied
kubectl exec dns-test -c dns-test -- curl -s http://service-python/env
{"message":"python v2","upstream":[{"message":"lua v1","response_time":0.3},{"message":"node v2","upstream":[{"message":"go v2","response_time":"0.02"}],"response_time":0.25}]}
kubectl exec dns-test -c dns-test -- curl -s http://service-js/
<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,shrink-to-fit=no"><meta name="theme-color" content="#000000"><link rel="manifest" href="/manifest.json"><link rel="shortcut icon" href="/favicon.ico"><title>React App</title><link href="/static/css/main.c17080f1.css" rel="stylesheet"></head><body><noscript>You need to enable JavaScript to run this app.</noscript><div id="root"></div><script type="text/javascript" src="/static/js/main.bc334d90.js"></script></body></html>
```

以上步骤创建了服务级别的授权，只允许指定服务调用，因此我们使用dns-test访问时只有service-python服务和service-js服务可以正常访问，因为服务service-python与服务service-js允许任何用户调用。

16）浏览器访问。

浏览器访问，服务正常

17）清理：

```powershell
kubectl delete -f kubernetes/dns-test.yaml
kubectl delete -f istio/security/rbac-config-on.yaml
kubectl delete -f istio/security/rbac-service-policy.yaml
kubectl delete -f istio/route/gateway-js-v1-mtls.yaml
kubectl delete -f istio/security/mtls-namespace-default-on.yaml
```
