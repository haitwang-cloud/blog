>本文是 [An introduction to Go Kubernetes' informers](https://macias.info/entry/202109081800_k8s_informers.md)的中文翻译版本，内容有删减

这篇文章介绍了[Kubernetes Go client library](https://pkg.go.dev/k8s.io/client-go/informers)工具，它主要用于在内存中保持集群资源的实时快照。

在代码示例中，我们导入了需要的包：

```golang
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
)
```

动机
----------

如果您的Go程序需要获取有关Kubernetes资源（例如服务、副本集、Pod等）的信息，您可以使用官方的Kubernetes Go `client`实例与Kubernetes APIServer进行交互：

```golang
// gets the information of a given pod in the default namespace
pod, err :=  client.CoreV1().Pods("default").
    Get(context.Background(), "pod-name", v1.GetOptions{})

// gets the information of all the currently existing pods in all the
// namespaces
pods, err := client.CoreV1().Pods(corev1.NamespaceAll).
    List(context.Background(), v1.ListOptions{})
```


然而，您可能希望最小化连接数来拉取数据。并减少获取资源的延迟，因此您可以使用`Watch`接口来监听Kubernetes资源的更改事件，依次保证内存是最新的资源：

```golang
// ignoring returned error on purpose
watcher, _ := client.CoreV1().Pods(corev1.NamespaceAll).
	Watch(context.Background(), metav1.ListOptions{})
for event := range watcher.ResultChan() {
    pod := event.Object.(*corev1.Pod)
    fmt.Printf("%v pod with name %s\n", event.Type, pod.Name)
}
```

上述代码的输出会是下面这样子

```shell
ADDED pod with name openshift-controller-manager-operator-6b4884d944-gbj2n
ADDED pod with name installer-3-ip-10-0-131-5.ec2.internal
ADDED pod with name kube-storage-version-migrator-operator-684c8fbd9-fw6p8
ADDED pod with name apiserver-86b697ffcb-424gl
ADDED pod with name kube-controller-manager-ip-10-0-131-5.ec2.internal
ADDED pod with name cluster-autoscaler-operator-558c76fc6-l4xwk
ADDED pod with name node-exporter-wjks6
```

为了确保在内存中保持集群中所有Pod的副本，您需要检查`event.Type`值（`Added`、`Deleted`、`Modified`等），然后相应地更新存储Pod数据的`Map`（例如按唯一字段索引，比如Pod的命名空间+名称）。

此时您需要为所有需要在内存中保持的资源编写相同的代码，包括：

* 管理内存存储和索引，以及必要时的并发访问控制。
* 建立与不同资源的监视连接，并管理重新连接的过程。

解决方案-->Informers
-----------------------

为了用最少的代码实现上述功能，Kubernetes Go库提供了名为`informers`的的功能，它们会持续监视您的Kubernetes资源更新事件（添加、删除、修改等），并在内存中保持它们的副本，可以通过给定的索引检索到这些资源副本。

您可以通过`informers`工厂为每种资源类型创建`informers`。例如，以下代码将创建一个Pods的`informers`：



```go
// resyncing in-memory copy each 10 minutes
factory := informers.NewSharedInformerFactory(client, 10*time.Minute)
podsInformer := factory.Core().V1().Pods().Informer()


```

下面的代码会启动上面初始化的_Informers_，（以及工厂创建的任何其他informer），并等待直到获取完整的Pod副本到内存中

```go
stopCh := make(chan struct{})
factory.Start(stopCh) // runs in background
factory.WaitForCacheSync(stopCh)
```

即使在等待缓存同步之后，工厂创建的所有`informers`仍会保持后台运行，以便在集群中的Pod发生任何更改时，内存中的数据都会被及时更新。您可以关闭`stopCh`通道来中断所有informers的后台执行。

默认情况下，针对有命名空间的资源的informers使用`namespace/name`字符串作为键来存储它们。您可以按以下方式按照命名空间和名称检索任何Pod：

```golang
// ignoring returned ok and err for brevity
podItem, _, _ := podsInformer.GetIndexer().GetByKey(namespace + "/" + name)
pod := podItem.(*corev1.Pod)
fmt.Println("The Pod IP is", pod.Status.PodIP)
```

请注意，由于Go语言目前缺乏泛型支持（详情请见[Go泛型提案](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)），在处理某些情况下仍需要使用`interface{}`类型。


现在假设除了通过名称访问Pod之外，您还希望通过IP地址对其进行索引。在启动Informer工厂之前，您可以添加一个新的索引器indexer。新的Pod索引器将接收一个`*Pod`实例，并返回一个`string`值列表，用作此Pod的索引。在我们的情况下，可以返回该Pod的IP地址列表。

```go
// arbitrary unique name for the new indexer
const ByIP = "IndexByIP"
func podIPIndexFunc(obj interface{}) ([]string, error) {
    pod, ok := obj.(*v1.Pod)
    if !ok {
        return nil, cache.ErrWrongType
    }

    // Extract the IP addresses from the Pod and return them as a list of strings.
    var ipList []string
    for _, ip := range pod.Status.PodIPs {
        ipList = append(ipList, ip.IP)
    }
    return ipList, nil
}
podsInformer.AddIndexers(map[string]cache.IndexFunc{ByIP: podIPIndexFunc})


```


当Informer启动后，任何新的Pod都将以两种方式进行索引：按其`namespace/name`进行索引，以及按其任意IP地址进行索引。

现在，要通过IP地址检索一个Pod，我们需要使用之前添加的新的`IndexByIP`索引：

```go
items, err := podsInformer.GetIndexer().ByIndex(ByIP, ip)
```

通常，如果没有传递IP的Pod，`items`数组将返回一个长度为零的数组，对于大多数现有的Pod，它将返回包含单个Pod的数组。

然而，对于特殊情况，比如[host-networked pods](https://www.alibabacloud.com/help/doc-detail/123997.htm)，它们共享相同的宿主机IP，将返回包含所有共享相同IP的Pod的数组。

此外，您可以告诉Informer工厂为其他资源创建Informers，这些Informers的工作方式类似于Pods informer。

```golang
replicaSetInformer := factory.Apps().V1().ReplicaSets().Informer()
servicesInformer := factory.Core().V1().Services().Informer()
```

Conclusions 结论
-----------
*   Go Kubernetes库中的Informers（[链接](https://pkg.go.dev/k8s.io/client-go/informers)）帮助我们减少代码冗余，避免手动维护Kubernetes资源的内存副本。
*   Informers库足够灵活，可以根据我们自己的用例进行扩展，例如按任意字段进行索引，甚至提供不同的存储层（本文未展示）。
*   一旦Go泛型推出，Informers API可以得到改进，提供更加简洁和类型安全的代码。