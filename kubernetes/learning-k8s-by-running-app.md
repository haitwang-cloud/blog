# 背景


Kubernetes（简称K8S）是一种开源的容器编排系统，用于自动化管理、部署和扩展容器化应用程序。K8S是云原生架构的核心组件之一，它可以帮助开发人员更轻松地构建和管理云原生应用程序。K8s还提供了许多高级功能，例如负载均衡、服务发现、自动伸缩、存储管理等，这些功能可以帮助开发人员更轻松地构建可靠的云原生应用程序。

虽然K8S是一个强大的容器编排系统，但它仍然存在一些缺点，包括以下几个方面：

1. **学习曲线较陡峭**：Kubernetes是一个非常复杂的系统，它需要掌握大量的概念和技术，包括容器、Pod、服务发现、负载均衡、存储、网络等。因此，对于初学者来说，学习曲线可能比较陡峭。
2. **部署和管理复杂度较高**：虽然Kubernetes提供了许多工具来简化部署和管理，但这些工具仍然需要较高的技术水平来使用。此外，由于Kubernetes是一个分布式系统，因此在规划、部署和管理方面都需要进行复杂的决策和操作。
3. **资源占用较高**：Kubernetes需要运行在一个较为庞大的基础设施上，因此它需要占用相对较高的资源，包括CPU、内存、存储等。此外，Kubernetes还需要运行多个组件和代理，这些组件和代理也会占用一定的资源。
4. **容易出现故障**：由于Kubernetes是一个复杂的分布式系统，因此它容易出现故障和问题。这些故障可能涉及各个方面，包括网络、存储、节点故障等。此外，由于Kubernetes的架构复杂，排查问题也可能需要较长的时间和技术支持。
5. **不适合小规模应用**：由于Kubernetes需要占用较高的资源和运行多个组件，因此它对于小规模应用来说可能过于复杂和冗余。对于一些简单的应用，使用Kubernetes可能并不划算

如果你是一个不了解 K8S的开发人员，那么本文将从具体的使用的角度来帮助你学习和理解K8S。在学习 Kubernetes 之前，先了解一些基础概念

无状态应用是指应用本身不依赖于任何状态信息。也就是说，无状态应用不会维护任何与用户或请求相关的信息，它仅仅根据输入的请求进行计算和处理，并将结果返回给客户端。无状态应用通常使用负载均衡器将请求分配到多个服务器上进行处理，从而实现高可用性和可扩展性。常见的无状态应用包括 Web 服务、RESTful API、静态网站等。

相对于无状态应用，有状态应用依赖于一定的状态信息来完成任务。有状态应用在处理请求时需要使用上下文信息，包括用户信息、会话状态、数据库连接状态等等。有状态应用通常需要使用持久化存储来保存状态信息，比如数据库、缓存、文件系统等。有状态应用不适合使用负载均衡器进行请求分发，因为请求需要在同一个服务器上处理，否则会出现状态不一致的问题。常见的有状态应用包括在线游戏、聊天应用、电子商务应用等。

需要注意的是，有状态应用和无状态应用并不是互相排斥的关系，而是根据应用的需求和特点来选择最合适的架构模式。有些应用可能既有无状态部分，也有有状态部分，需要使用混合的架构模式来实现。

## Load Balancing

负载均衡（Load Balancing）是一种在计算机网络中分配工作负载的技术，其主要目的是提高应用程序的可用性、性能和可伸缩性。当网络流量过大时，负载均衡可以通过将负载分配到多个服务器上来减轻单个服务器的压力，并确保所有服务器能够合理地处理请求。

负载均衡在现代应用程序和网络中起着至关重要的作用，特别是在高流量、高负载的情况下。它可以确保应用程序的可用性和可靠性，并提高用户体验。负载均衡技术在云计算和分布式系统中也得到广泛的应用，成为了构建高可用性、高性能和高可扩展性系统的重要基础。

### 客户端/服务端负载均衡

**客户端负载均衡**

![](./pics/part7-clientsidelb.png)

图片来源（[https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/](https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/)）

客户端负载均衡（Client-side Load Balancing）是一种在分布式系统中常用的负载均衡技术，它可以将请求从客户端分发到多个服务器，以提高系统的性能、可伸缩性和可用性。客户端负载均衡通常是通过在客户端应用程序中实现的，而不是在服务器端实现的。

在客户端负载均衡中，客户端应用程序会维护一个服务器列表，并根据负载均衡算法选择一个服务器来发送请求。负载均衡算法可以根据服务器的负载情况、网络延迟等因素来选择服务器，以实现最优的负载均衡效果。客户端应用程序还可以定期从服务发现中心获取服务器列表，并使用心跳检测等机制来监测服务器的可用性。常见的客户端负载均衡实现有 [Spring Cloud LoadBalancer](https://spring.io/guides/gs/spring-cloud-loadbalancer/) , [consul](https://www.consul.io/), [nacos](https://github.com/alibaba/nacos) 和[istio](https://istio.io/)

**服务端负载均衡**

![](./pics/part7-serversidelb.png)
图片来源（[https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/](https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/)）

服务端负载均衡（Server-side Load Balancing）是指通过在服务端引入负载均衡器（Load Balancer），将客户端请求分发到多个后端服务实例中，从而实现服务的高可用和高性能。通常，负载均衡器会根据不同的负载均衡算法（例如轮询、随机等）将客户端请求分配到后端的服务实例上。

服务端负载均衡器通常位于服务端的网络边缘，作为客户端和后端服务实例之间的中间层。它可以同时处理大量的客户端请求，并将请求转发到多个后端服务实例上，从而提高系统的处理能力和可靠性。同时，负载均衡器还可以实现一些高级功能，如故障检测、动态配置、流量控制等。常见的硬件负载均衡的厂家有 [F5 BIG-IP](https://www.f5.com/products/big-ip-services)，[Citrix NetScaler](https://docs.netscaler.com/en-us/citrix-adc.html)，[Barracuda Load Balancer](https://www.barracuda.com/products/application-cloud-security/load-balancer) 和 [A10 Networks Thunder](https://www.a10networks.com/products/thunder-adc/)

**服务端和客户端负载均衡对比**

服务端负载均衡和客户端负载均衡各有优缺点：

1. 负载均衡器的位置：服务端负载均衡器位于服务端，而客户端负载均衡器位于客户端。
2. 负载均衡器的数量：服务端负载均衡器通常是单个或少数几个，而客户端负载均衡器可以有多个，每个客户端都可以有自己的负载均衡器。
3. 服务实例列表的维护：服务端负载均衡器负责维护服务实例列表，而客户端负载均衡器需要从服务端获取服务实例列表或者自己维护服务实例列表。
4. 网络通信量：服务端负载均衡器需要将请求从客户端转发到服务实例，这可能会增加网络通信量。而客户端负载均衡器通常只需要在本地选择一个服务实例来处理请求，因此可以减少网络通信量。
5. 系统可用性：客户端负载均衡器无法动态地响应服务端的变化，一旦服务实例状态发生变化，客户端负载均衡器可能会选择到不可用的服务实例。而服务端负载均衡器可以及时响应服务实例的变化，从而提高系统的可用性。
6. 性能瓶颈：服务端负载均衡器可能成为性能瓶颈，而客户端负载均衡器通常可以在本地快速选择一个服务实例来处理请求，从而减少性能瓶颈的风险。

综上所述，服务端负载均衡和客户端负载均衡各有优缺点，需要根据具体业务场景和需求选择合适的负载均衡方式。服务端负载均衡适合服务实例数量较大、集中管理的场景，而客户端负载均衡适合服务实例数量较小、分散的场景。

### L4/L7 负载均衡

![](./pics/network-osi.png)

图片来源（《计算机网络第七版》谢希仁）
计算机网络体系结构

![](./pics/network-data.png)
OSI 七层模型和数据

图片来源（[https://icyfenix.cn/architect-perspective/general-architecture/diversion-system/load-balancing.html](https://icyfenix.cn/architect-perspective/general-architecture/diversion-system/load-balancing.html)）

**四层负载均衡**

L4负载均衡是在OSI模型的第四层（即传输层）上进行负载均衡的。这种负载均衡方式通常基于IP地址和端口号进行负载分配，将请求分配到后端服务器的方法通常是基于传输控制协议（TCP）连接的状态信息进行分配。这种方式通常速度较快，因为它不需要解析应用程序数据，但它对于不同的应用程序可能不是非常智能化，因为它无法解析HTTP等应用层协议的数据。

**七层负载均衡**

L7负载均衡是在OSI模型的第七层（即应用层）上进行负载均衡的。这种方式通过解析应用程序数据（如HTTP头部信息）来实现请求的分配，因此可以更智能地决定请求的分配方式。例如，可以根据请求的URL、请求头、请求体中的内容等信息来进行分配。这种方式通常需要较多的计算资源，因为它需要解析应用程序数据，但它可以更智能地分配请求并更好地适应应用程序层面的需求。

总的来说，L4负载均衡适用于需要快速响应的高吞吐量应用程序，而L7负载均衡适用于需要更智能的负载分配方式的应用程序。关于L4和L7更详细的介绍请参考[凤凰架构-负载均衡](https://icyfenix.cn/architect-perspective/general-architecture/diversion-system/load-balancing.html)。

### LVS负载均衡

LVS（Linux Virtual Server）是一种开源的负载均衡软件，它是在Linux操作系统上实现的。LVS可以通过多种负载均衡算法将客户端请求分发到多个后端服务器，从而提高系统的可用性和可扩展性。LVS主要有三种负载均衡技术：NAT、IP隧道和直接路由（Direct Routing）。

**NAT（Network Address Translation）**：在这种负载均衡模式下，LVS将客户端请求的源IP地址和端口号修改为LVS节点的IP地址和端口号，然后将请求转发到后端服务器。当后端服务器将响应发送回LVS节点时，LVS节点会将响应的目标IP地址和端口号修改为客户端的IP地址和端口号，然后将响应发送给客户端。

![](./pics/LVS-NAT.png)

图片来源（[https://cloud.tencent.com/developer/article/1619664](https://cloud.tencent.com/developer/article/1619664)）

**IP隧道（IP Tunneling）**：在这种负载均衡模式下，LVS会将客户端请求封装在一个IP隧道中，并将隧道中的请求发送到后端服务器。当后端服务器将响应发送回LVS节点时，LVS节点会将响应封装在一个IP隧道中，并将隧道中的响应发送给客户端。

![](./pics/LVS-TUN.png)

图片来源（[https://cloud.tencent.com/developer/article/1619664](https://cloud.tencent.com/developer/article/1619664)）

**直接路由（Direct Routing）**：在这种负载均衡模式下，LVS会将客户端请求的目标IP地址和端口号修改为后端服务器的IP地址和端口号，然后将请求发送到后端服务器。当后端服务器将响应发送回LVS节点时，LVS节点会将响应直接发送给客户端。

![](./pics/LVS-DR.png)

图片来源（[https://cloud.tencent.com/developer/article/1619664](https://cloud.tencent.com/developer/article/1619664)）

LVS提供了多种负载均衡算法，如轮询、最少连接数等。LVS还支持多种协议，如TCP、UDP、HTTP等。LVS是一个成熟、稳定的负载均衡软件，被广泛应用于互联网和企业内部的各种应用程序。关于LVS更加详细的介绍读者可查看 [LVS内核原理与LVS十种调度算法](https://cloud.tencent.com/developer/article/1619664)

## K8S相关概念

下面是关于k8s的基础组件的简要介绍：

### K8S基础概念

Pod：Pod 是 Kubernetes 中最小的可部署单元，它是一个可以包含一个或多个容器的逻辑主机。Pod 中的容器共享网络和存储资源，并可以通过 localhost 相互通信。Pod 旨在为容器提供一个较小的运行环境，将容器的生命周期、网络、存储等方面进行了抽象，使得开发人员更方便地管理和部署容器化应用

![](./pics/pod.png)

**Deployment**

Deployment是一种管理Pod副本的抽象概念，它可以用来定义一个应用程序的副本数量、Pod的模板以及更新策略等信息。Deployment通过创建和管理ReplicaSet来控制Pod的数量，并根据需要调整Pod的数量，以确保应用程序在集群中的副本数始终保持在指定的范围内，以满足应用的负载需求。

![](./pics/deployment.png)

**Service**

Service 是一种可以提供稳定网络端点的抽象，它可以将一个或多个 Pod 组合成一个服务。Service可以通过标签选择器来选择一组 Pod，并将它们作为一个服务暴露出去，从而让其他应用程序可以通过这个服务来访问这些 Pod。使用SVC，开发人员可以实现应用程序的发现和负载均衡，而无需了解后端 Pod 的具体信息

**ConfigMap 和 Secret**

ConfigMap 和 Secret 是 Kubernetes 中用于管理应用配置和敏感数据的资源。ConfigMap 可以管理应用的配置信息，例如数据库地址、端口号等；Secret 可以管理敏感数据，例如密码、API 密钥等。这些资源可以通过环境变量、配置文件、命令行参数等方式注入到应用中

![](./pics/secret.png)

想要深入了解 K8S的读者可以查看 [Kubernetes 基础教程](https://lib.jimmysong.io/kubernetes-handbook/architecture/perspective/#kubernetes-%E8%AE%BE%E8%AE%A1%E7%90%86%E5%BF%B5%E4%B8%8E%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F) 和 [Kubernetes 文档](https://kubernetes.io/docs/concepts/)

**kube-proxy**

前文提到Service为一组Pod提供了一个稳定的、虚拟的IP地址和DNS名称，具有负载均衡的功能。Service的负载均衡是基于L4负载均衡实现的。具体而言，Kube-proxy会根据请求的目标IP地址和端口，将请求转发到相应的后端Pod。这是一个基于TCP/UDP协议的转发过程，因此被称为四层负载均衡。在转发过程中，Kube-proxy会维护一些iptables或IPVS规则，根据这些规则来实现负载均衡和服务发现的功能。因此接下来介绍一些Kube-proxy的实现原理。

![](./pics/kube-proxy.jpeg)

Kube-proxy在Kubernetes集群的每个Node节点上运行，它通常作为一个独立的进程来运行，Kube-proxy进程可以通过 daemonset 或者 static pod 的方式运行。Kube-proxy通过监视Kubernetes API服务器上的Service和Endpoint对象的变化，自动更新集群节点上的iptables规则或ipvs规则，从而实现了负载均衡和服务发现的功能。

```go
+------------------------+
           |    Kubernetes API      |
           |    Server              |
           +-----------+------------+
                       |
                       | watch Service/Endpoint
                       v
           +-----------+------------+
           |        Kube-proxy      |
           | (iptables or IPVS mode)|
           +-----------+------------+
                       |
                       | update iptables/IPVS rules
                       v
           +-----------+------------+
           |       Pod Endpoint     |
           +------------------------+
```

Kube-proxy提供了三种代理模式：

- userspace模式：将Service的Cluster IP地址映射到一个本地未使用的端口，并在本地监听该端口，以接受请求。这种模式是最早的代理模式，但因其性能较差，已经不再推荐使用。
- iptables模式：将Service的Cluster IP地址映射到iptables规则，然后将请求转发到后端Pod。这种模式比userspace模式性能更好，同时也更稳定。

![](./pics/kube-proxy-iptables.svg)
图片来源（[https://docs.google.com/drawings/d/1MtWL8qRTs6PlnJrW4dh8135_S9e2SaawT410bJuoBPk/edit](https://docs.google.com/drawings/d/1MtWL8qRTs6PlnJrW4dh8135_S9e2SaawT410bJuoBPk/edit)）

ipvs模式：使用Linux内核的IPVS（IP Virtual Server）技术来实现负载均衡和服务发现。这种模式性能更好，但需要在节点上安装额外的软件包。

**PV&PVC**

在 Kubernetes (k8s) 中，PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) 是用来管理和使用持久化存储资源的两个重要概念。

PersistentVolume (PV) 是 Kubernetes 中的一种资源对象，它表示一个存储资源，例如云存储卷或本地磁盘。PV 可以是静态定义的，也可以是动态生成的。静态定义的 PV 在集群创建时就已经存在，而动态生成的 PV 则是根据管理员定义的存储类别动态创建的。

PersistentVolumeClaim (PVC) 是 Kubernetes 中的另一种资源对象，它表示对 PV 的请求。PVC 可以在 Pod 中声明，以请求一个或多个 PV 来存储应用程序的数据。当 Pod 挂载 PVC 时，Kubernetes 会自动将 PVC 绑定到一个可用的 PV 上。

使用 PV 和 PVC 的主要目的是让应用程序在不同的 Pod 和节点之间共享持久化存储。如果应用程序需要在多个 Pod 中共享数据，您可以使用 PVC 来请求一个 PV，并将 PV 挂载到多个 Pod 中。如果 PV 不再需要，您可以删除 PVC，Kubernetes 会释放 PV 并将其返回到可用资源池中。

# 应用

在了解完前面的基础内容以后，接下来我们开始通过从应用的角度把前面K8S的知识串起来理解，首先加入我们的目标是学习如何用k8s运行有状态应用

## 有状态应用-> MySQL
 MySQL 集群的部署是常见的有状态应用的例子,k8s官方也有响应的文档[运行一个有状态的应用程序](https://kubernetes.io/zh-cn/docs/tasks/run-application/run-replicated-stateful-application/#services),读者ke自行参考。本文将结合图文展开讲解。
 
 首先在K8S中部署MySQL集群，我们需要考虑以下几个问题：

- MySQL集群应该是一个“主从复制”（Maser-Slave Replication）的加购，只有 1 个主节点（Master），但有多个从节点（Slave）
- 从节点需要能水平扩展；
- 所有的写操作，只能在主节点上执行
- 读操作可以在所有节点上执行。

为了达成上面的的任务，需要用到的k8s 资源对象有 StatefulSet、PV/PVC、ConfigMap、Headless Service、ClusterIP Service.

![](./pics/k8s-mysql-arch.png)

图片来源（[K8s 应用管理之道 - 有状态服务](https://developer.aliyun.com/article/689685)）

通过上面的k8s的资源部署 MySQL集群的具体流程如下：

- 创建一个 PersistentVolume (PV) 对象来存储 MySQL 数据。可以使用已有的存储资源，也可以创建新的存储资源。

- 创建一个 PersistentVolumeClaim (PVC) 对象来请求上一步中创建的 PV。

- 创建一个 ConfigMap 对象来保存 MySQL 的配置信息。

- 创建一个 StatefulSet 对象来运行 MySQL 容器。在这个 StatefulSet 中，需要指定容器镜像、容器端口以及挂载 PV 和 ConfigMap 等配置。

- 创建两个 Service 对象来将 MySQL 容器暴露为 Kubernetes 集群内部或外部的服务，其中一个是名为mysql的 Headless Service, 另一个则是
mysql-read的普通 Service

下面是对新出现的概念的介绍
### Headless Service

Headless Service是Kubernetes中的一种服务类型，它与普通服务（ClusterIP）不同的是，Headless Service不会分配一个ClusterIP给Service，而是为每个Service的Endpoint创建DNS记录，使得客户端能够直接访问这些Endpoint。这些Endpoint通常是一个或多个Pod的网络地址，Headless Service将它们公开为一个有序列表。

Headless Service仍是一个标准 Service, 只不过，它的 clusterIP 字段的值 `None`
与普通服务不同，Headless Service没有ClusterIP和端口号，因此不能使用负载均衡和代理。因此，Headless Service通常需要结合其他服务使用，例如StatefulSet或Service Mesh

下面是对应的Service yaml配置
```
# 为 StatefulSet 成员提供稳定的 DNS 表项的无头服务（Headless Service）
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql

# 用于连接到任一 MySQL 实例执行读操作的客户端服务
# 对于写操作，你必须连接到主服务器：mysql-0.mysql
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
    readonly: "true"
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```


### StatefulSet

StatefulSet是Kubernetes中的一种控制器对象，它用于管理有状态应用程序的部署和管理。StatefulSet和Deployment不同之处在于，它管理的Pod有固定的标识符，例如名称和网络标识符，这些标识符随着Pod的重启和重新调度而保持不变。

StatefulSet的主要目的是支持有状态的应用程序，例如数据库或集群存储系统。有状态的应用程序通常要求稳定的网络标识符和持久性存储，以确保数据的一致性和可靠性。StatefulSet的工作原理是通过创建一组Pod来管理有状态应用程序，并为每个Pod分配唯一的标识符，例如名称或索引，从而提供稳定的网络标识符和持久性存储。

当一个Pod启动时，StatefulSet会为其分配一个唯一的名称，例如<StatefulSetName>-<Ordinal> ，其中<StatefulSetName>是StatefulSet的名称，<Ordinal>是Pod在StatefulSet中的序号。这些唯一的名称对于有状态应用程序非常重要，因为它们提供了一个固定的标识符，用于识别和访问Pod。

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
      app.kubernetes.io/name: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
        app.kubernetes.io/name: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
```

上面的yaml是 [mysql-statefulset.yaml](https://raw.githubusercontent.com/kubernetes/website/main/content/zh-cn/examples/application/mysql/mysql-statefulset.yaml) 的节选， 起结果是 Pods 名为 mysql-0、mysql-1 和 mysql-2。此外需要注意的是这个statefulset多了一个 serviceName=mysql 字段，这个字段的作用，就是告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 mysql 这个 Headless Service 来保证 Pod 的“可解析身份”。

以上就是如何基于 StatefulSet 在 k8s 中部署运维一套高可用 MySQL 服务，但过程相对复杂。对于新手用户很不友好，还需维护一套复杂的管理脚本。为了降低在 k8s 中部署复杂应用的门槛，诞生了 Kubernetes Operator，具体可以参考[mysql-operator-introduction](https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-introduction.html)



## 无状态应用  

首先假设我们的目标应用是常见的无状态应用，无状态应用是指在运行过程中不维护任何会话或状态信息的应用程序，每个请求都是独立的。这意味着应用程序可以随时被复制或迁移到不同的节点，而不会影响应用程序的功能或性能。

![](./pics/service.png)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 告知 Deployment 运行 2 个与该模板匹配的 Pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80


############################
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 80        
```
在上面的Service中我使用了 selector 字段来声明这个 Service 只代理携带了 app: nginx 标签的 Pod。并且，这个 Service 的 80 端口，代理的是 Pod 的 80 端口.

需要注意的是，只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。
### k8s proxy负载均衡实现原理

#### Iptables

```bash
# kubectl get svc nginx
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
nginx1   ClusterIP   10.106.224.41   <none>        8080/TCP   163m
```

此时在Node节点上访问该Service服务，首先流量到达的是OUTPUT链，这里我们只关心nat表的OUTPUT链：

```bash
# iptables-save -t nat | grep -- '-A OUTPUT'
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

该链跳转到`KUBE-SERVICES`子链中:

```bash
# iptables-save -t nat | grep -- '-A KUBE-SERVICES'
...
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.106.224.41/32 -p tcp -m comment --comment "default/kubernetes-bootcamp-v1: cluster IP" -m tcp --dport 8080 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.106.224.41/32 -p tcp -m comment --comment "default/kubernetes-bootcamp-v1: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-RPP7DHNHMGOIIFDC
```

我们发现与之相关的有两条规则：

- 第一条负责打标记 MARK`0x4000/0x4000`，后面会用到这个标记。
- 第二条规则跳到 `KUBE-SVC-RPP7DHNHMGOIIFDC` 子链。

其中 `KUBE-SVC-RPP7DHNHMGOIIFDC` 子链规则如下:

```bash
# iptables-save -t nat | grep -- '-A KUBE-SVC-RPP7DHNHMGOIIFDC'
-A KUBE-SVC-RPP7DHNHMGOIIFDC -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-FTIQ6MSD3LWO5HZX
-A KUBE-SVC-RPP7DHNHMGOIIFDC -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-SQBK6CVV7ZCKBTVI
-A KUBE-SVC-RPP7DHNHMGOIIFDC -j KUBE-SEP-IAZPHGLZVO2SWOVD
```

- 1/3 的概率跳到子链 `KUBE-SEP-FTIQ6MSD3LWO5HZX`,
- 剩下概率的 1/2，(1 - 1/3) * 1/2 == 1/3，即 1/3 的概率跳到子链 `KUBE-SEP-SQBK6CVV7ZCKBTVI`，
- 剩下 1/3 的概率跳到 `KUBE-SEP-IAZPHGLZVO2SWOVD`。

我们查看其中一个子链 `KUBE-SEP-FTIQ6MSD3LWO5HZX`规则:

```bash
# iptables-save -t nat | grep -- '-A KUBE-SEP-FTIQ6MSD3LWO5HZX'
...
-A KUBE-SEP-FTIQ6MSD3LWO5HZX -p tcp -m tcp -j DNAT --to-destination 10.244.1.2:8080
```

由此可见子链 `KUBE-SVC-RPP7DHNHMGOIIFDC`
 的功能就是按照概率均等的原则 DNAT 到其中一个 Endpoint IP，即 Pod IP，假设为 10.244.1.2，


### ipvs

```bash
$ ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.0.0.1:443 rr persistent 10800
  -> 192.168.0.1:6443             Masq    1      1          0
TCP  10.0.0.10:53 rr
  -> 172.17.0.2:53                Masq    1      0          0
UDP  10.0.0.10:53 rr
  -> 172.17.0.2:53                Masq    1      0          0
```

# 参考文献

- [Go Microservices, Part 7: Service Discovery and Load Balancing](https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/)
- [https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/](https://laptrinhx.com/go-microservices-part-7-service-discovery-and-load-balancing-2345614758/)
- [https://cloud.tencent.com/developer/article/1769822](https://cloud.tencent.com/developer/article/1769822)
- [http://docs.kubernetes.org.cn/117.html](http://docs.kubernetes.org.cn/117.html)
- [https://mp.weixin.qq.com/s/3LGRf9NMs9qkMlGvTqVAcA](https://mp.weixin.qq.com/s/3LGRf9NMs9qkMlGvTqVAcA)
- [https://kubernetes.feisky.xyz/concepts/components/kube-proxy](https://kubernetes.feisky.xyz/concepts/components/kube-proxy)
- [https://llussy.github.io/2019/12/12/kube-proxy-IPVS/](https://llussy.github.io/2019/12/12/kube-proxy-IPVS/)
- [https://developer.aliyun.com/article/727651](https://developer.aliyun.com/article/727651)
- [https://galaxyyao.github.io/2019/06/26/容器-10-Kubernetes实战-Service/](https://galaxyyao.github.io/2019/06/26/%E5%AE%B9%E5%99%A8-10-Kubernetes%E5%AE%9E%E6%88%98-Service/)
- [https://www.weave.works/blog/introduction-to-service-meshes-on-kubernetes-and-progressive-delivery](https://www.weave.works/blog/introduction-to-service-meshes-on-kubernetes-and-progressive-delivery)
- [https://in4it.io/kubernetes-secrets-management/](https://in4it.io/kubernetes-secrets-management/)
- [https://arthurchiao.art/blog/cracking-k8s-node-proxy/#2-kubernetes-node-proxy-model](https://arthurchiao.art/blog/cracking-k8s-node-proxy/#2-kubernetes-node-proxy-model)
- [K8s 应用管理之道 - 有状态服务](https://developer.aliyun.com/article/689685)
- [Understanding Persistent Volumes and PVCs in Kubernetes & OpenEBS](https://blog.mayadata.io/understanding-persistent-volumes-and-pvcs-in-kubernetes)