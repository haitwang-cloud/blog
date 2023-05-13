> 本文是 [leader-election-in-kubernetes-using-client-go](https://itnext.io/leader-election-in-kubernetes-using-client-go-a19cbe7a9a85)的中文翻译版本，内容有删减

如果您想了解 Kubernetes 中leader election的工作原理，那么希望本文能对您有所帮助。在本文中，我们将讨论高可用系统中leader election的概念，并探讨[kubernetes/client-go](https://github.com/kubernetes/client-go)库，以了解其在 Kubernetes 控制器中的应用。

近年来，“高可用性”一词因可靠系统和基础设施需求的增加而变得流行起来。在分布式系统中，高可用性通常涉及最大化运行时间和系统容错。高可用性中通常采用的一种做法是使用冗余来避免单点故障。为冗余做好系统和服务的准备工作可能只需要在负载均衡器后面部署更多的副本。虽然这样的配置对许多应用程序来说可能有效，但有些用例需要在副本之间进行仔细的协调才能使系统正确运行。

一个很好的例子是当一个 Kubernetes 控制器被部署为多个实例时。为了防止任何意外的行为，leader election过程必须确保在副本之间选出一个leader，并且该leader是唯一主动协调集群的实例。其他实例应该保持不活动，但随时准备接管leader实例的工作，以防其失败。

在 Kubernetes 中，leader election的过程很简单。它始于创建一个锁对象，leader会定期更新当前时间戳，以通知其他副本其领导权。这个锁对象可以是一个`Lease`，`ConfigMap`或者`Endpoint`，它还保存了当前leader的身份。如果leader在给定的时间间隔内未能更新时间戳，则认为它已经崩溃，此时非活动副本会竞争更新锁，以获取领导权。成功获取锁的pod将成为新的leader。

在我们开始写代码之前，我们来看一下这个过程是如何工作的。

首先，我们需要一个本地的Kubernetes集群。我将使用 [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/)，但是您可以随意选择一个本地的k8s发行版。



```shell
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:kubectl cluster-info --context kind-kindNot sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/

```

我们将使用的示例应用程序可以在[k8s-leader-election](https://github.com/mayankshah1607/k8s-leader-election)，它使用[kubernetes/client-go](https://github.com/kubernetes/client-go)实现leader election。让我们在我们的集群上运行应用程序：

```bash
# Setup required permissions for creating/getting Lease objects
$ kubectl apply -f rbac.yaml
serviceaccount/leaderelection-sa created
role.rbac.authorization.k8s.io/leaderelection-role created
rolebinding.rbac.authorization.k8s.io/leaderelection-rolebinding created# Create deployment
$ kubectl apply -f deploy.yaml
deployment.apps/leaderelection created

```

这将创建一个包含3个pod（副本）的deployment。如果您等待几秒钟，您应该会看到它们处于`Running`状态。

```bash
❯ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
leaderelection-6d5b456c9d-cfd2l   1/1     Running   0          19s
leaderelection-6d5b456c9d-n2kx2   1/1     Running   0          19s
leaderelection-6d5b456c9d-ph8nj   1/1     Running   0          19s

```

一旦您的pod运行起来，让我们尝试查看它们作为leader election过程的一部分创建的`Lease`锁对象。

```bash
$ kubectl describe lease my-leaseName:         my-lease
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  coordination.k8s.io/v1
Kind:         Lease
Metadata:
...
Spec:
  Acquire Time:            2021-10-23T06:51:50.605570Z
  Holder Identity:         leaderelection-56457b6c5c-fn725
  Lease Duration Seconds:  15
  Lease Transitions:       0
  Renew Time:              2021-10-23T06:52:45.309478Z


```

根据这个，我们当前的leader pod是`leaderelection-56457bc5c-fn725`。让我们通过查看我们的pod日志来验证这一点。

```bash
# leader pod
$ kubectl logs leaderelection-56457b6c5c-fn725I1023 06:51:50.605439       1 leaderelection.go:248] attempting to acquire leader lease default/my-lease...
I1023 06:51:50.630111       1 leaderelection.go:258] successfully acquired lease default/my-lease
I1023 06:51:50.630141       1 main.go:57] still the leader!
I1023 06:51:50.630245       1 main.go:36] doing stuff...# inactive pods
$ kubectl logs leaderelection-56457b6c5c-n857k
I1023 06:51:55.400797       1 leaderelection.go:248] attempting to acquire leader lease default/my-lease...
I1023 06:51:55.412780       1 main.go:60] new leader is %sleaderelection-56457b6c5c-fn725# inactive pod
$ kubectl logs leaderelection-56457b6c5c-s48kx
I1023 06:51:52.905451       1 leaderelection.go:248] attempting to acquire leader lease default/my-lease...
I1023 06:51:52.915618       1 main.go:60] new leader is %sleaderelection-56457b6c5c-fn725

```

尝试删除leader pod来模拟崩溃，并检查`Lease`对象的内容来判断是否选举出了新的leader 

这里的基本思想是使用分布式锁机制来决定哪个进程将成为leader。获取锁的进程将执行所需的任务。`main`函数是我们应用程序的入口。在这里，我们创建一个对锁对象的引用，并启动一个leader election循环。

```Go
func main() {
	var (
		leaseLockName      string
		leaseLockNamespace string
		podName            = os.Getenv("POD_NAME")
	)
	flag.StringVar(&leaseLockName, "lease-name", "", "Name of lease lock")
	flag.StringVar(&leaseLockNamespace, "lease-namespace", "default", "Name of lease lock namespace")
	flag.Parse()

	if leaseLockName == "" {
		klog.Fatal("missing lease-name flag")
	}
	if leaseLockNamespace == "" {
		klog.Fatal("missing lease-namespace flag")
	}

	config, err := rest.InClusterConfig()
	client = clientset.NewForConfigOrDie(config)

	if err != nil {
		klog.Fatalf("failed to get kubeconfig")
	}

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	lock := getNewLock(leaseLockName, podName, leaseLockNamespace)
	runLeaderElection(lock, ctx, podName)
}
```


我们首先解析`lease-name`和`lease-namespace`标志，以获取副本必须使用的锁对象的名称和命名空间。`POD_NAME`环境变量的值（[deploy.yaml](https://github.com/mayankshah1607/k8s-leader-election/blob/master/deploy.yaml#L26) manifest) 会用于标识`Lease`对象中的leader。最后，我们使用这些参数创建一个锁对象来启动leader election过程。

The `runLeaderElection` function is where we initiate the leader election loop by calling `RunOrDie` . We pass a `LeaderElectionConfig` to it:

`runLeaderElection`函数是我们通过调用`RunOrDie`来启动leader election循环的地方。我们向它传递一个`LeaderElectionConfig`：

```go
func runLeaderElection(lock *resourcelock.LeaseLock, ctx context.Context, id string) {
	leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
		Lock:            lock,
		ReleaseOnCancel: true,
		LeaseDuration:   15 * time.Second,
		RenewDeadline:   10 * time.Second,
		RetryPeriod:     2 * time.Second,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: func(c context.Context) {
				doStuff()
			},
			OnStoppedLeading: func() {
				klog.Info("no longer the leader, staying inactive.")
			},
			OnNewLeader: func(current_id string) {
				if current_id == id {
					klog.Info("still the leader!")
					return
				}
				klog.Info("new leader is %s", current_id)
			},
		},
	})
}
```

现在，让我们来看看client-go中`RunOrDie`的实现。

[https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#L218-L227](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#L218-L227)

它使用我们传递给它的`LeaderElectorConfig`创建一个`*LeaderElector`，并在其上调用`Run`方法：

[https://github.com/kubernetes/client-go/blob/56656ba0e04ff501549162385908f5b7d14f5dc8/tools/leaderelection/leaderelection.go#L200-L213](https://github.com/kubernetes/client-go/blob/56656ba0e04ff501549162385908f5b7d14f5dc8/tools/leaderelection/leaderelection.go#L200-L213)

这个方法负责运行leader election循环。它首先尝试获取锁（使用`le.acquire`）。在成功后，它运行我们之前配置的`OnStartedLeading`回调并定期更新租约。在无法获取锁时，它只是运行`OnStoppedLeading`回调并返回。

这里最重要的部分是`acquire`和`renew`方法中对`tryAcquireOrRenew`的调用，它包含了锁机制的核心逻辑。

### 优化锁（并发控制）[Optimistic locking (concurrency control)] 

leader election 过程利用Kubernetes的原子性，确保不会有两个实例同时获取到`Lease`中的锁，每次更新 `Lease`（续订或获取），Kubernetes 也会更新其上的 resourceVersion 字段。当另一个进程尝试同时更新 `Lease` 时，Kubernetes 会检查更新对象的 `resourceVersion` 字段是否与当前对象匹配——如果不匹配，则更新失败，从而防止并发问题！


在这篇文章中，我们介绍了leader election的概念以及它对分布式系统的高可用性的重要性。我们看了看Kubernetes是如何使用`Lease`锁来实现的，并尝试使用[kubernetes/client-go](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go)库。此外，我们还尝试了解Kubernetes如何使用原子操作和乐观锁方法来防止并发问题。


*   [https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/](https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/)
*   [https://medium.com/hybrid-cloud-hobbyist/leader-election-architecture-kubernetes-32600da81e3c](https://medium.com/hybrid-cloud-hobbyist/leader-election-architecture-kubernetes-32600da81e3c)
*   [https://carlosbecker.com/posts/k8s-leader-election/](https://carlosbecker.com/posts/k8s-leader-election/)
*   [https://taesunny.github.io/kubernetes/kubernetes-controllers-leader-election-with-go-library/](https://taesunny.github.io/kubernetes/kubernetes-controllers-leader-election-with-go-library/)