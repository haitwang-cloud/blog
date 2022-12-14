> 本文是[k3s vs k8s](https://www.civo.com/blog/k8s-vs-k3s)的中文翻译版本，内容有删减


## What is Kubernetes?

For those of you not in the know, Kubernetes is a "container orchestration platform". This effectively means taking your containers (everyone's heard of Docker by now, right?) and choosing which machine out of a group of them to run that container on.

对于那些不熟悉Kubernetes来说，Kubernetes其实是一个“容器编排平台”。这实际上意味着拿走你的容器（现在每个程序员都听说过[Docker](https://www.docker.com/)，对吧？）并从一组机器中决定哪台机器来运行该容器。

It also handles things like upgrades of your containers, so if you make a new release of your website, it will gradually launch containers with the new version and gradually kill off the old ones, usually over a minute or two.

它还处理诸如容器升级之类的事情，因此，如果您发布网站的新版本，它将逐渐启动具有新版本的容器，并逐渐杀死旧容器（参考[Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/) ），整个发布过程通常在一两分钟内。

## So what is K8s?
K8s is just an abbreviation of Kubernetes ("K" followed by 8 letters "ubernete" followed by "s"). However, normally when people talk about either Kubernetes or K8s, they are talking about the original upstream project, designed as a really highly available and hugely scalable platform by Google.

K8s 只是 Kubernetes 的缩写（“K”后跟 8 个字母“ubernete”，后跟“s”）。然而，通常当人们谈论 Kubernetes 或 K8s 时，他们谈论的其实是 Google 设计的一个高度可用且极具可扩展性的平台。

For example, here's an example of a [Kubernetes cluster handling a zero downtime update while performing 10 million requests per second](https://www.youtube.com/watch?v=9C6YeyyUUmI) on YouTube.

例如，这是一个YouTube上关于[利用Kubernetes 进行集群处理零停机更新，同时仍每秒执行 1000 万个请求](https://www.youtube.com/watch?v=9C6YeyyUUmI)的视频。

The problem is that while you can run Kubernetes on your local developer machine with [Minikube](https://kubernetes.io/docs/tasks/tools/), if you're going to run it in production you very quickly get in to the realm of"best practices" with advice like:

尽管你可以用 [Minikube](https://kubernetes.io/docs/tasks/tools/)在本地开发者机器上运行 Kubernetes，但如果你要在生产环境中运行它，你必须看看以下关于“最佳实践”的建议：


1.  Separate your masters from your nodes - your masters run the control plane and your nodes run your workload - and never the twain shall meet.
2.  Run etcd (the database for your Kubernetes state) on a separate cluster to ensure it can handle the load.
3.  Ideally have separate Ingress nodes so they can handle the incoming traffic easily, even if some of the underly nodes are slammed busy.

- 将你的主节点与其他节点分开: 主节点运行k8s控制平面，其他节点运行你的k8s工作负载,千万不要把它们混为一体
- 在单独的集群上运行 etcd（存储Kubernetes 状态的数据库），以确保它可以处理负载
- 理想情况下，应该配置与底层节点独立的Ingress节点，以便它们在底层节点繁忙时仍可以轻松处理传入流量

Very quickly this can get you to 3 x K8s masters, 3 x etcd, 2 x Ingress plus your nodes. So a realistic minimum of 8 medium instances before you even get to "how many nodes do I need for my site?".

通过上面的原则，我们可以推断出一个的节点配置方案是：3个K8s主节点；3个etcd；2个Ingress和其他的节点。

Don't misunderstand us, if you're running a production workload this is VERY sane advice. There's nothing worse than trying to debug a down production cluster that's overloaded, late on a Friday night!

别误解，如果您正在运行产线环境的工作负载，这是非常理智的建议。没有什么比在周五晚上尝试调试过载的下产线环境集群更糟糕的了！

## What is k3s and how is it different from k8s?

K3s is designed to be a single binary of less than 40MB that completely implements the Kubernetes API. In order to achieve this, they removed a lot of extra drivers that didn't need to be part of the core and are easily replaced with add-ons.

K3s 被设计成了一个小于 40MB 的单个二进制文件，它完全复用了了 Kubernetes API。为了实现这一目标，K3s设计者删除了许多不需要成为核心并容易被附加组件替换的驱动程序。

K3s is a fully CNCF (Cloud Native Computing Foundation) certified Kubernetes offering. This means that you can write your YAML to operate against a regular "full-fat" Kubernetes and they'll also apply against a k3s cluster.

K3s 是 CNCF（云原生计算基金会）认证的 Kubernetes 产品。这意味着你的 YAML即可以在常规的Kubernetes上运行，同时也可以 k3s 集群上运行。

Due to its low resource requirements, it's possible to run a cluster on anything from 512MB of RAM machines upwards. This means that we can allow pods to run on the master, as well as nodes.

由于K3s对资源要求低，甚至可以在 512MB 以上的 RAM 计算机上运行集群。这意味着我们可以允许 Pod 在主节点和其他节点上运行。

And of course, because it's a tiny binary, it means we can install it in a fraction of the time it takes to launch a regular Kubernetes cluster! We generally achieve sub-two minutes to launch a k3s cluster with a handful of nodes, meaning you can be deploying apps to learn/test at the drop of a hat.

当然由于K3s是一个很小的二进制文件，这意味着它的启动速度则非常快，通常可以在不到两分钟的时间里启动具有少量节点的 k3s 集群，这意味着您可以立即着手部署应用程序来学习/测试。

Both its reputation and adopton is growing rapidly too, with over 17k Github stars since its launch in early 2019, whilst it was recently crowned the number 1 new developer tool of 2019 by Stackshare

K3s的声誉和采用率也在迅速增长，自2019年初推出以来，K3s的[Github Star超过17000](https://github.com/k3s-io/k3s)，而最近它被Stackshare评为[2019年第一大新开发者工具](https://stackshare.io/posts/top-developer-tools-2019)。

## Is k3s the same as k8s, just better?
Well, pretty much. When most people think of Kubernetes they think of containers automatically being brought up on other nodes (if the node dies), of load balancing between containers, of isolation and rolling deployments - and all of those advantages are the same between "full-fat" K8s vs k3s.

其实差不多。当大多数人想到 Kubernetes 时，他们会想到容器自动在其他节点上启动（如果节点死亡）、容器之间的负载平衡、隔离和滚动部署——所有这些优势在K8s 与 k3s 之间都是相同的。

So what are the differences in using k3s?

那么使用 k3 有什么区别呢？

Primarily, the default database in single control-plane k3s clusters is SQLite. The performance is great for small clusters, but can need replacing with something more powerul such as etcd, MySQL or PostgreSQL if a larger cluster is required! Fortunately k3s supports all of them (whereas upstream Kubernetes only supports etcd)!

首先，单个控制平面 k3s 集群中的默认数据库是 SQLite。性能对于小型集群来说很棒，但如果需要更大的集群，可能需要用更强大的东西替换，例如 etcd、MySQL 或 PostgreSQL！幸运的是，k3s 支持所有这些（而上游 Kubernetes 只支持 etcd）！

The other real difference only really applies if you're one of the bigger cloud providers where you may have a lot of your extensions already in the upstream Kubernetes source code, because k3s removes all of those extensions and relies on standard interfaces such as the Container Storage Interface (CSI) for implementing them. This has no effect on end-customers though, only on the service provider itself.

另一个真正的区别则只针对较大的云服务提供商，假设你可能已经在上Kubernetes 源代码中有很多扩展，因为 k3s 删除了所有这些扩展并依赖于标准接口[CSI](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/) 来实现它们。不过这对终端客户没有影响，只对云服务提供商本身有影响。

## Should I choose k3s or k8s?

The simple answer is k3s - it's fully CNCF certified as a conforming Kubernetes cluster, it's lighter weight and trimmed back - and who doesn't want to get rid of excess bloat in your software!

简单的答案是 k3s - 它是完全被 CNCF 认证为符合 Kubernetes 集群，更轻量级，并且经过修剪 - 谁不想摆脱软件中多余的臃肿！

More question about kubernetes? Check our [Kubernetes vs Docker article](https://www.civo.com/blog/kubernetes-vs-docker-a-comprehensive-comparison).

更多关于 kubernetes 的问题？查看我们的 [Kubernetes vs Docker 文章](https://www.civo.com/blog/kubernetes-vs-docker-a-comprehensive-comparison)。