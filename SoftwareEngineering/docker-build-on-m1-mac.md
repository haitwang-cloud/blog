> [How To Build Docker Images For Apple Silicon (Aka M1 Chip)](https://jitsu.com/blog/multi-platform-docker-builds) 的中文翻译版本,内容有删减

背景（Background）
--------------------------

苹果不久前发布了基于M1芯片的新MacBook。与之前所有基于英特尔的Apple笔记本电脑不同，M1具有不同的架构 - `arm64`而不是英特尔的“amd64”。


新的架构意味着大多数软件产品应该重新编译,至少对于用编译语言编写的软件必须这样做。该规则适用于主要用`Go`写的项目。然而，这并不是100%正确的：苹果不遗余力地使旧的编译代码与新架构兼容，[Roseta 2](https://en.wikipedia.org/wiki/Rosetta_(software))是用来达成这个目标的主要工具.

Docker 也试图迎头赶上 [the M1 support came out a few weeks ago](https://docs.docker.com/docker-for-mac/apple-silicon/).但是，事情并不像预期那样无缝运行：

Docker依靠 [qemu](https://www.qemu.org/) 在M1芯片上模拟英特尔的架构。然而事实是 QEMU有时不能很好地工作：_在 Apple Silicon 机器上尝试运行基于 Intel 的容器可能会崩溃，因为 QEMU 有时无法运行容器。例如文件变更 API [File System Events](https://developer.apple.com/documentation/coreservices/file_system_events)（例如 inotify）在 QEMU 仿真下不起作用 [...]。  因此，我们建议您在 Apple Silicon 机器上运行 ARM64 容器。与基于 Intel 的容器相比，这些容器也更快且使用更少的内存_。


如何为`arm64`打包镜像（How to build an image for `arm64`）
---------------------------------------------

幸运的是，[Docker 已经宣布](https://www.docker.com/blog/multi-platform-docker-builds/) 一项实验性功能 **[buildx](https://github.com/docker/buildx)**，它可以为特定架构（arm64、amd64 等）构建映像。

### Step 1.开启实验功能（ Enable experimental features0

如果您使用的是 Mac 版 Docker Desktop，您只需在 Docker CLI 上启用实验性功能 —— 编辑 **~/.docker/config.json** 添加以下内容：
```json
{
  ...
  "experimental": "enabled"
}

```

之后重新启动您的 Docker Desktop

如果您使用的是 Linux，只需运行以下命令：

```shell
docker run --privileged --rm docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64
```

并使用 `service docker restart` 重启 Docker

### Step 2. 创建一个新的构建器实例(Create a new builder instance)

创建一个新的构建器实例：

```shell
docker buildx create --use
```

It enables multiple builds in parallel. Make sure that new builder instance is created:

它支持多个并行构建。 可以用以下命令来确保创建了新的构建器实例：


Output:

```
$ docker buildx ls
NAME/NODE     DRIVER/ENDPOINT       STATUS  PLATFORMS
mystifying_bell * docker-container
  mystifying_bell0 unix:///var/run/docker.sock inactive
default      docker
  default     default           running linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

### Step 3. Build multi-architecture image[#](#step-3-build-multiarchitecture-image)

Build your multi-architecture image:

通过以下命令构建您的多架构映像：

```shell
docker buildx build --platform linux/amd64,linux/arm64 -t company/image_name .
```
**platform** 标志指定将为哪些平台构建 Docker 映像。 目前Docker 支持 10 个平台，但可能你应该只使用 `linux/amd64` (Intel) 和 `linux/arm64` (M1)：

![](https://jitsu.com/img/blog/multi-platform-docker-builds/docker-hub-result.png)

Docker Hub result

Reference
--------------------------
- [buildx](https://github.com/docker/buildx)
- [qemu](https://www.qemu.org/)