
> 本文是[NOTICE: Containers losing access to GPUs with error: "Failed to initialize NVML: Unknown Error"](https://github.com/NVIDIA/nvidia-container-toolkit/issues/48)的中文翻译版本，内容有删减，亲测该方法有效。
> 
# 如何解决 `Failed to initialize NVML: Unknown Error` 问题

## 概述

在某些特定的情况下，我们发现k8s容器可能会突然从最初连接到的GPU上分离。我们已经确定了这个问题的根本原因，并确定了可能发生这种情况的受影响环境。在本文档的末尾提供了受影响环境的解决方法，直到发布适当的修复为止。

## 问题总结


我们发现当使用container来管理GPU工作负载时，用户container可能会突然失去对GPU的访问权限。这种情况发生在使用systemd来管理容器的cgroups时，当触发重新加载任何包含对NVIDIA GPU的引用的Unit文件时（例如，通过执行`systemctl daemon-reload`）。

当你的container失去对GPU的访问权限时，你可能会看到类似于以下错误消息：

```shell
Failed to initialize NVML: Unknown Error
```
一旦发生上述 ⬆️，就需要手动删除受影响的container，然后重新启动它们。

当container重新启动（手动或自动，取决于是否使用容器编排平台），它将重新获得对GPU的访问权限。

此问题的根源在于，最近版本的`runc`要求在`/dev/char`下面为注入到容器中的任何设备节点提供符号链接。不幸的是，NVIDIA设备并没有这些符号链接，NVIDIA GPU驱动也没有（当前）提供自动创建这些链接的方法

## 受影响的环境


**如果你使用`runc`并在高级容器运行时（CRI）启用`systemd cgroup`管理的环境，那么你可能会受到这个问题的影响**

如果你没有使用`systemd`来管理`cgroup`，那么它就不会受到这个问题的影响。
下面是可能会受影响的的环境的详尽列表:
### 使用`containerd`/`runc`的Docker环境
#### 特定条件
- 启用了`systemd`的`cgroup`驱动程序（例如，在`/etc/docker/daemon.json`中设置了参数`"exec-opts": ["native.cgroupdriver=systemd"]`）
- 使用了更新的`Docker`版本，其中`systemd cgroup`管理被默认设置的（即，在Ubuntu 22.04上）。

>`Note`如果你要检查`Docker`是否使用`systemd cgroup`管理，运行以下命令（下面的输出表示启用了`systemd cgroup`驱动程序）
```shell
 $ docker info  
 ...  
 Cgroup Driver: systemd  
 Cgroup Version: 1
```
### 使用`containerd`/`runc`的Kubernetes环境
#### 特定条件
- 在`containerd`配置文件（通常位于：`/etc/containerd/config.toml`）中设置`SystemdCgroup = true`，如下所示：
  
```shell
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"
...
SystemdCgroup = true
```
> Note:如果你想要检查`containerd`是否使用`systemd cgroup`管理，运行以下命令（下面的输出表示启用了`systemd cgroup`驱动程序）
```shell
$ sudo crictl info  
...  
"runtimes": {
    "nvidia": {
        "runtimeType": "io.containerd.runc.v2",
        ...
        "options": {
          "BinaryName": "/usr/local/nvidia/toolkit/nvidia-container-runtime",
          ...
          "ShimCgroup": "",
          "SystemdCgroup": true
```
### 使用`cri-o`/`runc`的Kubernetes环境
#### 特定条件
在`cri-o`配置文件中启用了`systemd`的`cgroup_manager`（通常位于：`/etc/crio/crio.conf或/etc/crio/crio.conf.d/00-default`），如下所示（使用`OpenShift`的示例）：
```shell
[crio.runtime]
...
cgroup_manager = "systemd"

hooks_dir = [
"/etc/containers/oci/hooks.d",
"/run/containers/oci/hooks.d",
"/usr/share/containers/oci/hooks.d",
]
```

Note: `Podman`环境默认使用`crun`，并且除非刻意将`runc`配置为要使用的底层容器运行CRI时，否则不会受到这个问题的影响。
## 如何检查你的环境是否受到影响

您可以按照以下步骤确认您的系统是否受到影响。您可以这些步骤来确认错误是否可以复现。
### Docker环境

运行一个测试的`container`：
```shell
$ docker run -d --rm --runtime=nvidia --gpus all \
    --device=/dev/nvidia-uvm \
    --device=/dev/nvidia-uvm-tools \
    --device=/dev/nvidia-modeset \
    --device=/dev/nvidiactl \
    --device=/dev/nvidia0 \
    nvcr.io/nvidia/cuda:12.0.0-base-ubuntu20.04 bash -c "while [ true ]; do nvidia-smi -L; sleep 5; done"  

bc045274b44bdf6ec2e4cc10d2968d1d2a046c47cad0a1d2088dc0a430add24b
```
> Note 请确保你测试的`container`挂载不同的设备（如上所示）。这是确定此特定问题的必要条件。

如果您的系统有超过1个GPU，请在上述命令后附加额外的 --device 挂载。拥有2个GPU的系统的例子：
```shell
$ docker run -d --rm --runtime=nvidia --gpus all \
    ...
    --device=/dev/nvidia0 \
    --device=/dev/nvidia1 \
    ...
```
此时可以查看`container`的log
```shell
$ docker logs bc045274b44bdf6ec2e4cc10d2968d1d2a046c47cad0a1d2088dc0a430add24b

GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
```
然后使用下面的命令来触发 `daemon-reload`
```shell
$ sudo systemctl daemon-reload
```
此时再次查看`container`的log, 我们可以复现这个问题
```shell
$ docker logs bc045274b44bdf6ec2e4cc10d2968d1d2a046c47cad0a1d2088dc0a430add24b

GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
GPU 0: Tesla K80 (UUID: GPU-05ea3312-64dd-a4e7-bc72-46d2f6050147)
Failed to initialize NVML: Unknown Error
Failed to initialize NVML: Unknown Error
```
### K8s 环境
运行一个测试的`Pod`：
```shell
$ cat nvidia-smi-loop.yaml

apiVersion: v1
kind: Pod
metadata:
  name: cuda-nvidia-smi-loop
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda
    image: "nvcr.io/nvidia/cuda:12.0.0-base-ubuntu20.04"
    command: ["/bin/sh", "-c"]
    args: ["while true; do nvidia-smi -L; sleep 5; done"]
    resources:
      limits:
        nvidia.com/gpu: 1


$ kubectl apply -f nvidia-smi-loop.yaml  
 
pod/cuda-nvidia-smi-loop created
```
此时可以查看`Pod`的log
```shell
$ kubectl logs cuda-nvidia-smi-loop

GPU 0: NVIDIA A100-PCIE-40GB (UUID: GPU-551720f0-caf0-22b7-f525-2a51a6ab478d)
GPU 0: NVIDIA A100-PCIE-40GB (UUID: GPU-551720f0-caf0-22b7-f525-2a51a6ab478d)
```
然后使用下面的命令来触发 `daemon-reload`
```shell
$ sudo systemctl daemon-reload
```
此时再次查看`Pod`的log, 我们可以复现这个问题
```shell
$ kubectl logs cuda-nvidia-smi-loop

GPU 0: NVIDIA A100-PCIE-40GB (UUID: GPU-551720f0-caf0-22b7-f525-2a51a6ab478d)
GPU 0: NVIDIA A100-PCIE-40GB (UUID: GPU-551720f0-caf0-22b7-f525-2a51a6ab478d)
Failed to initialize NVML: Unknown Error
Failed to initialize NVML: Unknown Error
```

## Workarounds

下面的解决方法适用于独立的docker环境和k8s环境（按照推荐顺序👍提供多个选项；最推荐👍的选项在最上面）：
### Docker环境

#### 使用`nvidia-ctk`工具:

NVIDIA Container Toolkit v1.12.0包含了一个在`/dev/char`中为所有可能的NVIDIA设备节点创建符号链接的工具，这些节点对于在容器中使用GPU是必要的。该工具可以如下运行：

```shell
sudo nvidia-ctk system create-dev-char-symlinks \
    --create-all
```

这个命令应该在每个节点上的启动时运行，这样容器中使用GPU时就不会出现问题。在运行这个命令之前，需要确保NVIDIA驱动的内核模块已经加载。

一个简单的`udev`规则可以强制执行这一点，如下所示：

```shell
# This will create /dev/char symlinks to all device nodes
ACTION=="add", DEVPATH=="/bus/pci/drivers/nvidia", RUN+="/usr/bin/nvidia-ctk system 	create-dev-char-symlinks --create-all"
```

安装此规则的推荐的地方是：`/lib/udev/rules.d/71-nvidia-dev-char.rules`


在使用`NVIDIA GPU Driver Container`的情况下，必须指定驱动程序安装的路径。在这种情况下，命令应该修改为：

```shell
sudo nvidia-ctk system create-dev-symlinks \
        --create-all \
        --driver-root={{NVIDIA_DRIVER_ROOT}}
```

`{{NVIDIA_DRIVER_ROOT}}`是NVIDIA GPU Driver container安装NVIDIA GPU驱动程序并创建NVIDIA设备节点的路径。

#### 隐式地禁用Docker中的`systemd cgroup`管理

设置`/etc/docker/daemon.json`文件中的参数`"exec-opts": ["native.cgroupdriver=cgroupfs"]`并重启docker。


#### 降级docker.io

降级到`docker.io`包，其中`systemd`不是默认的`cgroup`管理器（当然不要覆盖）。

### K8s 环境

#### GPU Operator

你可以使用`GPU Operator`的`22.9.2`以及以上版本，它会自动修复集群中所有K8s节点上的问题（此修复已经集成到[gpu-operator-validator](https://github.com/NVIDIA/gpu-operator/tree/master/validator)中，当新Node部署或每次Node重启时都会运行）。

#### k8s-device-plugin

对于直接使用k8s-device-plugin的场景来说（即不通过`GPU Operator`使用），你可以安装如上一节所述的`udev`规则来解决此问题。。如果你使用了container来安装driver的话，请请确保传递正确的`{{NVIDIA_DRIVER_ROOT}}`。详情请参考[configuration-option-details in k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin?tab=readme-ov-file#configuration-option-details)

#### 在`containerd`或`cri-o`隐式地禁用`systemd cgroup`

从cri-o配置文件中删除参数`cgroup_manager = "systemd"`（通常位于：`/etc/crio/crio.conf`或`/etc/crio/crio.conf.d/00-default`），然后重启cri-o。

#### 降级`containerd`

降级到`containerd.io`包的一个版本，其中`systemd`不是默认的`cgroup`管理器（当然不要覆盖）。

## 相关问题链接🔗
- ["Failed to initialize NVML: Unknown Error" after random amount of time](https://github.com/NVIDIA/nvidia-docker/issues/1671)
- [NOTICE: Containers losing access to GPUs with error: "Failed to initialize NVML: Unknown Error" ](https://github.com/NVIDIA/nvidia-docker/issues/1730)
- [Updating cpu-manager-policy=static causes NVML unknown error](https://github.com/NVIDIA/nvidia-docker/issues/966#issuecomment-610928514)