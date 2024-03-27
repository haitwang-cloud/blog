# haitwang-cloud blog

🌠欢迎来到haitwang-cloud博客🌠,
这里有作者在各技术领域的学习心得🧠
包括:
- 💻 Elasticsearch
- 🌈 Golang
- 🎮 GPU
- 📡 网络技术
- ⚙️ 软件开发
- 🤖 Kubernetes

> 其中 📖 为作者原创文章, 📓 为作者翻译文章  

## Elasticsearch 🐘
* [📖] [No Elasticsearch Node Available for olivere/elastic](./ElasticSearch/olivere/elastic.md)

## GPU 🎮
* [📓] [如何在基于 Rocky Linux 的 Kubernetes 上安装带有 A100 的 NVIDIA GPU Operator](./GPU/how-to-install-nvidia-gpu-operator-with-a100-on-kubernetes-base-rocky-linux.md)
* [📓] [如何使用 NVIDIA MPS 提高 Kubernetes 中的 GPU 利用率](./GPU/how-to-increase-gpu-utilization-in-kubernetes.md)
* [📓] [如何解决"Failed to initialize NVML: Unknown Error"](./GPU/failed-to-initialize-nvml-unknow-error.md)

## Golang 👨🏻‍💻
* [📓] [为什么在 Go 中使用 TestMain 进行测试？](./Golang/TestMain.md)
* [📓] [Go 中比较切片 (数组) 的 3 种方法](./Golang/compare-slice.md)
* [📓] [Go init 函数](./Golang/the-golang-init-func.md)
* [📓] [go.mod 文件中的直接和间接依赖](./Golang/direct-indirect-dependency-module-go.md)
* [📓] [Golang sync.Map](./Golang/Go-sync-Map.md)
* [📓] [Go 1.18 中的新功能](./Golang/go-version-118-release-new.md)
* [📓] [Golang 中的有效错误处理](./Golang/error-hanlde.md)
* [📓] [Golang中的值传递与引用传递](./Golang/golang-pass-by-value-vs-pass-by-reference.md)
* [📓] [使用Go管理多个Go版本](./Golang/managing-multiple-go-versions-with-go.md)
* [📓] [如何升级Golang的依赖](./Golang/how-to-upgrade-golang-dependencies.md)
* [📓] [Golang中正确使用条件变量sync.Cond](./Golang/go-sync-cond.md)
* [📓] [教程：开始使用模糊测试](./Golang/go-fuzz-testing.md)
* [📓] [基于表驱动的单元测试](./Golang/Table-driven-unit-tests.md)
* [📓] [Golang内存泄漏](./Golang/Golang-Memory-Leaks.md)
* [📓] [LeakProf: 轻量级在线Goroutine泄漏检测](./Golang/leakprof-featherlight-in-production-goroutine-leak-detection.md)
* [📓] [开始使用Go插件包](./Golang/getting-started-with-golang-plugins.md)
* [📓] [Cobra使用指南](./Golang/cobra-user-guide.md)

## Network 🌐

* [📓] [什么是BGP？ | BGP路由解析](./NetWork/what-is-bgp.md)
* [📓] [如何在Linux中使用ipset命令](./NetWork/how-to-use-ipset-command-in-linux.md)
* [📓] [QUIC的发展之路](./NetWork/the-road-to-quic.md)
* [📓] [在OSX上的Tcpdump和Wireshark](./NetWork/tcp-dump-in-OSX.md)
* [📓] [根证书与中间证书的区别](./NetWork/root-certificates-intermediate.md)
* [📓] [如何调试Istio Upstream重置502 UPE（旧503 UC）](./NetWork/how-to-debug-istio-upstream-reset.md)
* [📖] [在一个k8s集群中如何安装多个istio控制平面](./NetWork/how-to-install-multi-istio-control-plane.md)
* [📖] [在一个k8s集群中如何在多个istio环境中构建应用](./NetWork/build-app-under-multi-istio.md)

## Software Development 🛠️
* [📓] [软件开发中的上游和下游](./SoftwareEngineering/Upstream%3Adownstream/upstream-downstream.md)
* [📓] [JSON Patch and JSON Merge Patch](./SoftwareEngineering/json-patch-vs-merge-patch.md)
* [📓] [即时编译 (Just in Time)](./SoftwareEngineering/just-in-time-compilation-explained.md)
* [📓] [如何在Apple 芯片也称为M1芯片）上构建Docker镜像](./SoftwareEngineering/docker-build-on-m1-mac.md)

## Kubernetes 🕸️

* [📓] [K3s与K8s的区别是什么?](./kubernetes/k8s-vs-k3s.md)
* [📓] [Kubernetes headless Service](./kubernetes/headLess-svc.md)
* [📖] [从应用开发者的角度来学习K8S](./kubernetes/learning-k8s-by-running-app.md)
* [📓] [使用client-go在Kubernetes中进行leader election](./kubernetes/leader-election-in-kubernetes-using-client-go.md)
* [📓] [用k8sgpt-localai解锁Kubernetes的超能力](./kubernetes/k8sgpt-operater.md)
* [📖] [K8S的Endpoints和EndpointSlice的利弊对比](./kubernetes/k8s-svc-endpoint-slice.md)
* [📓] [在K8s controller-runtime和client-go中实现速率限制](./kubernetes/controller-runtime-client-go-rate-limiting.md)
* [📓] [Kubernetes' informers的介绍](./kubernetes/k8s_informers.md)
* [📖] [k8s **Affinity与 taint/toleration的区别解释**](./kubernetes/diff-of-Affinity-and-taint.md)
* [📖] [k8s默认的调度器工作机制和策略](./kubernetes/k8s-schedule-road-path.md)
* [📖] [通过 k8s Cloud-Provider 来学习如何设计一个 Controller](./kubernetes/k8s-cloud-provider.md)
* [📓] [使用Helm的Tpl函数来引用Values文件中的值](./kubernetes/using-the-helm-tpl-function-to-refer-values-in-values-files.md)
* [📖] [Client-go 中的label selector 引起的 **CPU Throttling**问题](./kubernetes/oom-killed-by-client-go-label-select.md)
* [📓] [深入了解Kubernetes控制器部分二 – 对象存储（object stores）和索引器（indexers）](./kubernetes/object-stores-and-indexers.md)
* [📓] [Introducing Bare Metal Kubernetes: what you need to know](./kubernetes/introducing-bare-metal-kubernetes-what-you-need-to-know.md)
* [📖] [Kubernetes中Containerd与Kubelet的Cgroup驱动配置冲突解决方案](./kubernetes/OCI-runtime-cgroupsPath-issue.md)