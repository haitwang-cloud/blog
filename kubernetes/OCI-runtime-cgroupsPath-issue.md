> 本文是针对作者遇到的`OCI runtime create failed: expected cgroupsPath to be of format \"slice:prefix:name\" for systemd cgroups, got \"/kubepods/burstable/..."`的问题总结


## 问题总结

### 问题描述
在特定的k8s node上不能通过`containerd`启动pod,pod的状态一直是`ContainerCreating`,通过`kubectl describe pod`查看pod的状态,发现如下错误:
```shell
OCI runtime create failed: runc create failed: expected cgroupsPath to be of format "slice:prefix:name" for systemd cgroups
```
### k8s集群信息
- k8s版本: `v1.26.13`
- containerd版本: `1.6.24`
- Linnux kernel版本: `6.6.20-amd64`
- Linux发行版: `Garden Linux 1443.0`
- kubeProxyVersion: `v1.26.13`
- kubeletVersion: `v1.26.13`
  
### 问题分析

此问题是因为`kubelet`配置为使用`cgroupfs` cgroup驱动程序，而`containerd`配置为使用`sytemd` cgroup驱动程序。

### 解决方法

为了解决上面的问题，可以从以下两种方式中选择一种：
- 让`containerd`使用`cgroupfs`驱动程序，需要从`/etc/containerd/config.toml`中删除`SystemdCgroup = true`行。
- 让`kubelet`使用`systemd`驱动程序，需要将[`KubeletConfiguration`](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)中的`cgroupDriver`设置为`"systemd"`。

## 扩展阅读

### 查看`Kubelet`配置

`Kubelet`的配置文件通常位于`/var/lib/kubelet/`位置，可以通过查看该文件来确认`Kubelet`的cgroup驱动程序配置。关于其他CRIs的配置文件位置可以参考[Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)。


```shell
~ ls -lh /var/lib/kubelet/
total 48K
-rw-r--r--  1 root root 1.5K Mar 27 01:35 ca.crt
d---------  2 root root 4.0K Mar 27 01:35 config
-rw-------  1 root root   62 Mar 27 01:35 cpu_manager_state
drwxr-xr-x  2 root root 4.0K Mar 27 03:06 device-plugins
-rw-------  1 root root 2.4K Mar 27 01:35 kubeconfig-real
-rw-------  1 root root   61 Mar 27 01:35 memory_manager_state
drwxr-xr-x  2 root root 4.0K Mar 27 01:35 pki
drwxr-x---  6 root root 4.0K Mar 27 03:06 plugins
drwxr-x---  2 root root 4.0K Mar 27 01:36 plugins_registry
drwxr-x---  2 root root 4.0K Mar 27 01:35 pod-resources
drwxr-x--- 27 root root 4.0K Mar 27 06:23 pods
drwxr-xr-x  2 root root 4.0K Mar 27 01:35 volumeplugins
```

### 查看`containerd`配置

`containerd`的配置文件通常位于`/etc/containerd/config.toml`位置，可以通过查看该文件来确认`containerd`的cgroup驱动程序配置。关于其他CRIs的配置文件位置可以参考[Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)。

```shell
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true
```

### 查看Linux node上的`Cgroup`版本

cgroup 版本取决于正在使用的 Linux 发行版和操作系统上配置的默认 cgroup 版本。 要检查你的发行版使用的是哪个 cgroup 版本，请在该节点上运行 stat -fc %T /sys/fs/cgroup/ 命令：

 ```shell
    $ stat -fc %T /sys/fs/cgroup/
```

如果输出为 cgroup2fs，则表示你用的是cgroup v2。
如果输出为 tmpfs，则表示你用的是cgroup v1。

关于cgroup v1和cgroup v2的更多信息，请参考[Control Groups (Cgroups)](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#control-groups-cgroups)。

## 参考链接
- [Kubernetes: OCI runtime create failed: expected cgroupsPath to be of format "slice:prefix:name" for systemd cgroups](https://github.com/containerd/containerd/issues/4857)
- [Set Kubelet Parameters Via A Configuration File](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)
- [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#systemd-cgroup-driver)