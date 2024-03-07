# 在多个istio版本的k8s环境中构建应用
本教程主要基于Istio官方文档中的 [Bookinfo Application](https://istio.io/latest/docs/examples/bookinfo/)，在多个istio版本的k8s环境中构建应用。

### 前提条件
请先参考上一篇文章[如何在同一个k8s cluster中安装多个istio版本](https://github.com/haitwang-cloud/blog/blob/main/NetWork/how-to-install-multi-istio-control-plane.md)配置好相应的istio环境，
接下来请将对应的[bookinfo.yaml
](https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml)文件下载到本地。

### 开始安装bookInfo应用
通过下面的命令来在`l7-traffic-alpha` namespace中安装bookInfo应用
```shell
➜ ~ k apply -f book-info.yaml -n l7-traffic-alpha
# service/details created
# serviceaccount/bookinfo-details created
# deployment.apps/details-v1 created
# service/ratings created
# serviceaccount/bookinfo-ratings created
# deployment.apps/ratings-v1 created
# service/reviews created
# serviceaccount/bookinfo-reviews created
# deployment.apps/reviews-v1 created
# deployment.apps/reviews-v2 created
# deployment.apps/reviews-v3 created
# service/productpage created
# serviceaccount/bookinfo-productpage created
# deployment.apps/productpage-v1 created
```
通过下面命令来查看对应的svc和pod是否已经创建成功,我们可以发现对应的svc已经创建成功
```shell
➜ ~ k get svc -n l7-traffic-alpha

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   100.111.56.251   <none>        9080/TCP   2m25s
productpage   ClusterIP   100.110.107.55   <none>        9080/TCP   2m16s
ratings       ClusterIP   100.109.32.83    <none>        9080/TCP   2m23s
reviews       ClusterIP   100.110.71.73    <none>        9080/TCP   2m21s
```
但是我们发现对应的pod并没有创建成功，需要查看一下root cause
```shell
➜ ~ k get deploy -n l7-traffic-alpha


NAME             READY   UP-TO-DATE   AVAILABLE   AGE
details-v1       0/1     0            0           3m40s
productpage-v1   0/1     0            0           3m31s
ratings-v1       0/1     0            0           3m38s
reviews-v1       0/1     0            0           3m35s
reviews-v2       0/1     0            0           3m34s
reviews-v3       0/1     0            0           3m33s
➜ ~ k get rs -n l7-traffic-alpha
NAME                        DESIRED   CURRENT   READY   AGE
details-v1-ccbdcf56         1         0         0       4m15s
productpage-v1-6c4f8467b9   1         0         0       4m6s
ratings-v1-96d98f9f6        1         0         0       4m13s
reviews-v1-57c85f47fb       1         0         0       4m10s
reviews-v2-64776cb9bd       1         0         0       4m9s
reviews-v3-5c8886f9c6       1         0         0       4m8s
```
Deploy对应的RS都已经创建成功，但是pod并没有创建成功，通过查看对应的pod的日志，我们发现是webhook的配置错误导致的，我们需要在`MutatingWebhookConfiguration`中修复它
```shell
➜ ~ k describe rs details-v1-ccbdcf56 -n l7-traffic-alpha
Name:           details-v1-ccbdcf56
Namespace:      l7-traffic-alpha
Selector:       app=details,pod-template-hash=ccbdcf56,version=v1
Labels:         app=details
                pod-template-hash=ccbdcf56
                version=v1
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/details-v1
Replicas:       0 current / 1 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           app=details
                    pod-template-hash=ccbdcf56
                    version=v1
  Service Account:  bookinfo-details
  Containers:
   details:
    Image:        docker.io/istio/examples-bookinfo-details-v1:1.18.0
    Port:         9080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                    From                   Message
  ----     ------        ----                   ----                   -------
  Warning  FailedCreate  113s (x16 over 4m37s)  replicaset-controller  Error creating: Internal error occurred: failed calling webhook "rev.namespace.sidecar-injector.istio.io": failed to call webhook: Post "https://istiod-1-18-5.istio-system.svc:443/inject?timeout=10s": service "istiod-1-18-5" not found
```
我们发现是service`istiod-1-18-5`这个istio-system`NameSpace`没有找到，但是由于我们的istio是多版本的，所以我们需要在`istio-canary`这个`NameSpace`中找到对应的service

```shell
(⎈|garden-ait--dl-devops-external:default garden-ait)➜  ➜ ~ k get svc -n istio-canary
NAME                   TYPE           CLUSTER-IP        EXTERNAL-IP      PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      100.109.23.65     <none>           80/TCP,443/TCP                                                               105m
istio-ingressgateway   LoadBalancer   100.110.197.23    10.237.149.244   15021:30331/TCP,80:32232/TCP,443:32582/TCP,31400:31519/TCP,15443:32673/TCP   117m
istiod-1-18-5          ClusterIP      100.109.220.197   <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        105m
istiod-revision        ClusterIP      100.111.11.61     <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        117m
```

这个`MutatingWebhookConfiguration`用于在生产环境中部署Canary deployments。它会给部署的Pods添加一个指定Pod的revision的annotation。这个annotation是Istio用来判断Pod是否是Canary deployment的一部分。具体来说，istio-revision-tag-prod-canary会给Pods添加以下annotation:
```shell
➜ ~ k get MutatingWebhookConfiguration istio-revision-tag-prod-canary -o yaml | grep "namespace: istio-system" -C 2
    service:
      name: istiod-1-18-5
      namespace: istio-system
      path: /inject
      port: 443
--
    service:
      name: istiod-1-18-5
      namespace: istio-system
      path: /inject
      port: 443
```
接下来我们需要把这个`MutatingWebhookConfiguration` `istio-revision-tag-prod-canary`中的NameSpace改为`istio-canary`

我们可以通过下面的命令来修改对应的`MutatingWebhookConfiguration`,并且删除现在的replicaset
```shell
➜ ~ k delete rs -n l7-traffic-alpha --all
replicaset.apps "details-v1-5f4d584748" deleted
replicaset.apps "productpage-v1-564d4686f" deleted
replicaset.apps "ratings-v1-686ccfb5d8" deleted
replicaset.apps "reviews-v1-86896b7648" deleted
replicaset.apps "reviews-v2-b7dcd98fb" deleted
replicaset.apps "reviews-v3-5c5cc7b6d" deleted
```
接下来我们再去查看一下pod的状态，我们会发现对应的pod已经创建成功并且处于running状态
```shell
➜ ~ k get pod -n l7-traffic-alpha
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-5f4d584748-xww9x      2/2     Running   0          100s
productpage-v1-564d4686f-wkxlr   2/2     Running   0          100s
ratings-v1-686ccfb5d8-vv8z7      2/2     Running   0          100s
reviews-v1-86896b7648-phv27      2/2     Running   0          100s
reviews-v2-b7dcd98fb-fs2sc       2/2     Running   0          99s
reviews-v3-5c5cc7b6d-65257       2/2     Running   0          99s
```
同上，我们也需要在 `istio-prod` namespace和`l7-traffic-alpha` namespace中安装对应的bookinfo应用

```shell
➜ ~ k apply -f book-info.yaml -n l7-traffic-alpha
```
### 开始配置Istio的ServiceMesh的CRD

首先从[bookinfo-gateway.yaml](https://github.com/istio/istio/blob/master/samples/bookinfo/gateway-api/bookinfo-gateway.yaml)下载对应的yaml文件，
接下来进行安装

```shell
➜ ~  k apply -f bookinfo-gateway.yml -n l7-traffic-alpha
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
(⎈|garden-ait--multi-istio-01-external:default garden-ait)➜  k get Gateway -n l7-traffic-alpha
NAME               AGE
bookinfo-gateway   21s
➜ ~  k get virtualservice -n l7-traffic-alpha
NAME       GATEWAYS               HOSTS                       AGE
bookinfo   ["bookinfo-gateway"]   ["bookinfo-gateway.test"]   31s
```

如果此时我们查看对应的`istiod`的日志，我们会发现对应的gateway/VirtualService已经被push到了`istiod`中

```shell
➜ ~ k logs istiod-1-19-3-64857b74d8-b5d5g -n istio-canary -f | grep "bookinfo"
2023-10-27T06:00:55.741297Z	info	ads	Push debounce stable[32] 1 for config Gateway/l7-traffic-alpha/bookinfo-gateway: 100.213717ms since last change, 100.213544ms since last push, full=true
2023-10-27T06:00:56.524033Z	info	ads	Push debounce stable[33] 1 for config VirtualService/l7-traffic-alpha/bookinfo: 100.164161ms since last change, 100.164004ms since last push, full=true
```

set the `INGRESS_HOST` /`INGRESS_PORT`/`GATEWAY_URL` variables for accessing the gateway.

接下来我们配置下面三个环境变量 `INGRESS_HOST` /`INGRESS_PORT`/`GATEWAY_URL`

```shell
➜ ~  export INGRESS_NAME=istio-ingressgateway
➜ ~  export INGRESS_NS=istio-canary
➜ ~  export INGRESS_HOST=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
➜ ~ export INGRESS_PORT=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
➜ ~  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

通过`CURL`命令来访问对应的host，我们会发现对应的resp是200

```shell
➜ ~  curl -s -I -HHost:bookinfo-gateway.test "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 27 Oct 2023 06:33:28 GMT
content-type: text/html; charset=utf-8
content-length: 5293
x-envoy-upstream-service-time: 1172
```

> 请注意，您使用-H标志将Host HTTP标头设置为“bookinfo-gateway.test”。这是必需的，因为您的入口网关配置为处理“bookinfo-gateway.test”，但在测试环境中，您没有该主机的DNS绑定，并且只是将请求发送到入口IP。
config the gateway/VirtualService for the `l7-traffic-beta` namespace

### 补充作业

接下来在`l7-traffic-beta` namespace中，我们使用另外一个基于nginx的sample应用来测试对应的gateway/VirtualService

```yml
# sample-gateway.yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nginx-sample-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio.io/rev: 1-18-5
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
  name: nginx-sample
spec:
  hosts:
  - "*"
  gateways:
  - nginx-sample-gateway
  http:
  - match:
    - uri:
        prefix: /nginx
    - sourceLabels:
        app: nginx-sample-client    
    route:
    - destination:
        host: nginx-sample-v1
        port:
          number: 80
  - match:
    - uri:
        prefix: /echo
    - sourceLabels:
        app: echo-sample-client        
    route:
    - destination:
        host: echo-sample-v1
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: nginx-sample-ds
spec:
  host: nginx-sample
  subsets:
  - name: nginx
    labels:
      app: nginx-sample

---

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo-sample-ds
spec:
  host: echo-sample
  subsets:
  - name: echo
    labels:
      app: echo-sample
```

```shell
➜  ~ k apply -f sample-gateway.yml -n l7-traffic-beta
```
此时我们可以通过下面的命令来查看对应的istiod的日志，我们会发现对应的gateway/VirtualService已经被push到了`istiod`中
```shell
➜ ~  k logs istiod-1-18-5-6f9cf7dcc4-84jvl -n istio-prod -f | grep "nginx-sample"
2023-10-27T07:55:28.421941Z	info	ads	Push debounce stable[38] 1 for config Gateway/l7-traffic-beta/nginx-sample-gateway: 100.246865ms since last change, 100.246675ms since last push, full=true
2023-10-27T07:55:29.179175Z	info	ads	Push debounce stable[39] 1 for config VirtualService/l7-traffic-beta/nginx-sample: 100.238736ms since last change, 100.238506ms since last push, full=true
```

if we curl the URL for the GW, the resp shoule be 200
如果此时我们通过`CURL`命令来访问对应的host，我们会发现对应的resp是200
```shell
➜ ~  k get svc -n istio-prod istio-ingressgateway
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   100.104.31.241   a8790f018666a43ad92a1add681cbfd0-2083997433.eu-west-1.elb.amazonaws.com   15021:30816/TCP,80:31241/TCP,443:32336/TCP   22h
➜ ~  curl -s -I -HHost:nginx-sample-gateway.test "http://a8790f018666a43ad92a1add681cbfd0-2083997433.eu-west-1.elb.amazonaws.com:80/"
HTTP/1.1 200 OK
server: istio-envoy
date: Fri, 27 Oct 2023 09:00:16 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 24 Oct 2023 13:46:47 GMT
etag: "6537cac7-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 3
```

#### 结论
本文档介绍了如何在同一个 Kubernetes 集群中部署两个版本的 Istio (1.18.5 和 1.19.3)，并分别在两个命名空间 (`l7-traffic-alpha` 和`l7-traffic-beta`) 中配置 Istio 网关和虚拟服务来控制流量。

### 参考文档
- [https://istio.io/latest/docs/examples/bookinfo/](https://istio.io/latest/docs/examples/bookinfo/)
- [Blue/Green Deployment with Istio: Match Host Header and sourceLabels for Pod to Pod Communication](https://www.lisenet.com/2021/blue-green-deployment-with-istio-match-host-header-and-sourcelabels-for-pod-to-pod-communication/)
- [Ingress Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)
- [Resource Annotations](https://istio.io/latest/docs/reference/config/annotations/)
- [https://github.com/istio/istio/issues/16126](https://github.com/istio/istio/issues/16126)