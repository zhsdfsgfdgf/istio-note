通过Denier适配器可以实现简单的服务访问控制，通过设置条件，可以拒绝部分服务间的访问

```yaml
 1 apiVersion: config.istio.io/v1alpha2
 2 kind: denier
 3 metadata:
 4   name: deny-handler
 5 spec:
 6   status:
 7     code: 7
 8     message: Not allowed
 9 ---
10 apiVersion: config.istio.io/v1alpha2
11 kind: checknothing
12 metadata:
13   name: deny-request
14 spec:
15 ---
16 apiVersion: config.istio.io/v1alpha2
17 kind: rule
18 metadata:
19   name: deny-service-node-v2
20 spec:
21   match: destination.labels["app"] == "service-go" && source.labels["app"]
          == "service-node" && source.labels["version"] == "v2"
22   actions:
23   - handler: deny-handler.denier
24     instances:
25     - deny-request.checknothing
```
第1~8行定义了名为deny-handler的denier适配器，code定义了请求被拒绝后返回的gRPC的响应码，code为7表示没有访问权限，与HTTP中的403响应码意义相同，message定义了请求被拒绝时返回的错误信息（面向开发者）。

第10~14行定义了名为deny-request的checknothing实例，直接返回空数据，一般用于测试。

第16~26行定义了名为deny-service-node-v2的rule，该规则表明，当访问service-node服务的v2版本时，无法得到service-go服务的正常响应。

【实验】

1）创建测试Pod：

```powershell
kubectl apply -f kubernetes/dns-test.yaml
```

2）创建Denier规则：

```powershell
kubectl apply -f istio/security/policy-rule-deny-simple.yaml
```

3）访问service-node服务：

```powershell
$ kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
{"message":"node v1","upstream":[{"message":"go v1","response_time":"0.50"}]}
$ kubectl exec dns-test -c dns-test -- curl -s http://service-node/env
{"message":"node v2","upstream":[]}
```

当访问service-node服务的v1版本时，可以得到service-go服务的正常响应；当访问service-node服务的v2版本时，无法得到service-go服务的正常响应。这正好符合我们设置的访问权限策略，说明我们设置的Denier适配器规则已经生效。

4）清理：

```powershell
kubectl delete -f kubernetes/dns-test.yaml
kubectl delete -f istio/security/policy-rule-deny-simple.yaml
```
