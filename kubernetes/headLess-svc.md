> 本文由 [headless-services-in-kubernetes](https://www.goglides.dev/bkpandey/headless-services-in-kubernetes-what-why-and-how-39fl)的中文翻译版本，内容有删减

Kubernetes headless Service是一个没有专用负载均衡器的service。这种类型的Service 通常用于**有状态的**应用程序。例如数据库，这些应用要求必须为每个实例维护一致的网络标识。如果客户端需要连接所有 Pod，则无法使用常规 Kubernetes的 ClusterIP Service来完成此操作。Service将无法将每个连接转发到随机选择的容器。

### 常规的Service是如何工作的？（How does Regular Service Object Works?）

接下来我们通过下面的yaml配置文件来创建一个常规的Kubernetes ClusterIP Service。

```Shell
cat <<EOF | kubectl apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: normal-nginx
  labels:
    app: normal-nginx # Deployment labels to match with replicaset labels and pods labels
spec:
  replicas: 3
  selector:
    matchLabels:
      app: normal-nginx # Replicaset to manage pods with labels
  template:
    metadata:
      labels:
        app: normal-nginx # Pods labels
    spec:
      containers:
        - name: nginx
          image: nginx
---
apiVersion: v1 # v1 is the default API version. It is the latest version of the Core API.
kind: Service # Service is a collection of Pods that are running on a host.
metadata:
  name: normal-nginx # name of the service
  labels:
    app: normal-nginx # label for the service
spec:
  selector:
    app: normal-nginx # label for the service
  ports:
  - port: 80 # port to expose
    targetPort: 80 # container port
    protocol: TCP # protocol to use for the port mapping ( example TCP, UDP, or SCTP)
  type: ClusterIP # type of service to create, ClusterIP is the default. Others are NodePort, LoadBalancer, and ExternalName.
EOF
```
Output:  
```Shell
deployment.apps/normal-nginx created
service/normal-nginx created
``
接下来我们通过下面的命令来验证我们的Service是否创建成功。

```Shell 
kubectl get svc -o wide | grep normal-nginx
```

Output:  

```Shell
normal-nginx   ClusterIP  10.96.141.251  <none>    80/TCP  48s   app=normal-nginx
```

然后是 `normal-nginx` pods的验证。

```Shell
kubectl get pods -o wide | grep normal-nginx
```

Output:  
```Shell
normal-nginx-d98bff4-jk65q    1/1   Running  0     74s   10.244.2.20  kind-worker2  <none>      <none>
normal-nginx-d98bff4-sbzf9    1/1   Running  0     74s   10.244.1.19  kind-worker  <none>      <none>
normal-nginx-d98bff4-wbj7q    1/1   Running  0     74s   10.244.2.19  kind-worker2  <none>      <none>

```

Now if you run `nslookup` to `normal-nginx` service object you will see following output:  
现在，如果您运行 `nslookup` 到 `normal-nginx` 服务对象，您将看到以下输出：

```Shell
kubectl run -it --rm debug --image=tutum/dnsutils --restart=Never nslookup normal-nginx
```

Output:  
```Shell
Server:     10.96.0.10
Address:    10.96.0.10#53

Name:   normal-nginx.default.svc.cluster.local
Address: 10.96.141.251
pod "debug" deleted
```
`nslookup normal-nginx` 的结果是 `ClusterIP` 地址。

### Headless Service是如何工作的（How does Headless Service Object Works?

对于客户端来说，要连接到所有的pod，它需要知道每个pod的IP地址。一种选择是让客户端调用Kubernetes API server，并通过API调用获取pod和IP地址的列表。但是，使用APIserver并不理想，因为您应该始终努力减少API调用以提高性能。另一种选择是使用Headless Service，它将为您提供一个静态IP地址列表，您可以使用该列表连接到pod。


[![](https://www.goglides.dev/images/K3aFm27H4vF5tsNCAVipQdzUHLGkEHB_ZuiF0zsv0ZY/w:880/mb:500000/ar:1/aHR0cHM6Ly93d3ct/Z29nbGlkZXMtZGV2/LnMzLmFtYXpvbmF3/cy5jb20vdXBsb2Fk/cy9hcnRpY2xlcy8y/bXVpOWZsZHhqOXpm/b2wzdDl5Ny5wbmc)](https://www.goglides.dev/images/K3aFm27H4vF5tsNCAVipQdzUHLGkEHB_ZuiF0zsv0ZY/w:880/mb:500000/ar:1/aHR0cHM6Ly93d3ct/Z29nbGlkZXMtZGV2/LnMzLmFtYXpvbmF3/cy5jb20vdXBsb2Fk/cy9hcnRpY2xlcy8y/bXVpOWZsZHhqOXpm/b2wzdDl5Ny5wbmc)


Kubernetes 允许客户端查找服务中pod的IP地址。通常，当您查找服务地址时，您会得到一个单独的IP地址：负载均衡器或ClusterIP的地址。如果您使用Headless Service时，那么Kubernetes将为您提供所有pod的IP地址列表。客户端可以通过执行DNS A记录查找来查找服务的所有pod的IPs。然后，客户端可以使用该信息连接到一个、多个或所有pod。

Headless service在进行单个pod的健康检查时也很有用。使用常规Service时，健康检查是在负载均衡器上执行的，该负载均衡器仅将流量转发到pod。这意味着pod可能不健康，但负载均衡器永远不会知道，因为它只是转发流量。使用headless service时，健康检查是在单个pod上执行的，因此您可以确保流量路由到健康的pod。

接下来，我们来运行相同的测试。首先，创建部署对象和headless service。在服务规范中将clusterIP字段设置为None意味着Kubernetes将不会为服务分配集群IP地址，您可以按照以下方式执行：


```Shell
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: headless-nginx
  labels:
    app: headless-nginx # Deployment labels to match with replicaset labels and pods labels
spec:
  replicas: 3
  selector:
    matchLabels:
      app: headless-nginx # Replicaset to manage pods with labels
  template:
    metadata:
      labels:
        app: headless-nginx # Pods labels
    spec:
      containers:
        - name: nginx
          image: nginx
---
apiVersion: v1 # v1 is the default API version. It is the latest version of the Core API.
kind: Service # Service is a collection of Pods that are running on a host.
metadata:
  name: headless-nginx # name of the service
  labels:
    app: headless-nginx # label for the service
spec:
  selector:
    app: headless-nginx # label for the service
  clusterIP: None # this is will point directly to pods, nslookup will rerun pod ips
  ports:
  - port: 80 # port to expose
    targetPort: 80 # container port
    protocol: TCP # protocol to use for the port mapping ( example TCP, UDP, or SCTP)
  type: ClusterIP # type of service to create, ClusterIP is the default. Others are NodePort, LoadBalancer, and ExternalName.
EOF

```
你会得到下面的输出

```Shell
deployment.apps/headless-nginx created
service/headless-nginx created
```

继续验证

```Shell
kubectl get svc -o wide
```

Output:  
```Shell
NAME       TYPE    CLUSTER-IP  EXTERNAL-IP  PORT(S)  AGE   SELECTOR
headless-nginx  ClusterIP  None     <none>    80/TCP  43s   app=headless-nginx
```

```Shell
kubectl get pods -o wide
```

Output:  

```Shell
NAME               READY  STATUS  RESTARTS  AGE  IP      NODE      NOMINATED NODE  READINESS GATES
headless-nginx-5795b5d5db-9sv8k  1/1   Running  0     71s  10.244.2.18  kind-worker2  <none>      <none>
headless-nginx-5795b5d5db-d5xmn  1/1   Running  0     71s  10.244.1.18  kind-worker  <none>      <none>
headless-nginx-5795b5d5db-dgwzq  1/1   Running  0     71s  10.244.1.17  kind-worker  <none>      <none>

```

接下来运行`nslookup`测试，如下所示

```Shell
kubectl run -it --rm debug --image=tutum/dnsutils --restart=Never nslookup headless-nginx
```

Output:  

```Shell
Server:     10.96.0.10
Address:    10.96.0.10#53

Name:   headless-nginx.default.svc.cluster.local
Address: 10.244.1.18
Name:   headless-nginx.default.svc.cluster.local
Address: 10.244.1.17
Name:   headless-nginx.default.svc.cluster.local
Address: 10.244.2.18

pod "debug" deleted

```
可以看到，`nslookup`命令为headless service返回了三个IP地址。

### headless servicey应用场景（Some of the use cases of headless service）

* 数据库复制和集群：数据库中的数据在两个或多个数据库之间同步。MongoDB，Couchbase等服务使用headless服务来集群和同步数据库之间的数据。
* 创建有状态服务：StatefulSet是用于管理有状态应用程序的工作负载API对象。它管理一组Pod的部署和缩放，并提供有关这些Pod的顺序和唯一性的保证。Headless服务使用稳定的主机名和IP地址暴露StatefulSet的Pod。RabbitMQ或Kafka（或任何其他消息代理服务）可以使用有状态集来部署到Kubernetes。
* 容器编排:不建议将容器编排工具（如marathon，kube-scheduler等）直接暴露给互联网，但headless服务可以帮助您在内部暴露这些工具，并使它们从外部世界保持安全。
* TLS终止：使用headless服务，我们可以在任何节点上终止TLS，然后可以在未加密的连接上进行通信。
* 服务发现：正如我们在示例中看到的，headless服务可以用于执行`nslookup`命令并获取Pod的IP地址。

### headless service的缺点(Drawbacks of headless service)

Headless service也存在一些缺点。主要的一个是它可能更难调试，因为您必须查看单个pod，而不是单个负载均衡器。这通常不是一个大问题，因为大多数服务都有某种日志记录，可以帮助您调试问题。
