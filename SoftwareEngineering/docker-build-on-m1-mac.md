> [How To Build Docker Images For Apple Silicon (Aka M1 Chip)](https://jitsu.com/blog/multi-platform-docker-builds) 的中文翻译版本,内容有删减

Background
--------------------------

Apple released a new MacBooks based on a M1 chip a while ago. Unlike all previous Apple laptops (which were Intel based), M1 has a different architecture — `arm64` instead Intel's `amd64`.

苹果不久前发布了基于M1芯片的新MacBook。与之前所有基于英特尔的Apple笔记本电脑不同，M1具有不同的架构 - `arm64`而不是英特尔的“amd64”。

New architecture means that the most software products should be repackaged: at least ones written in compiled-languages. It applies to Jitsu which is written mostly in `Go`. However, it's not 100% true: Apple went to great lengths to make old compiled code compatible with new architecture. The piece of software that makes is possible is called [Roseta 2](https://en.wikipedia.org/wiki/Rosetta_(software)).

新的架构意味着大多数软件产品应该重新编译,至少对于用编译语言编写的软件必须这样做。该规则适用于主要用`Go`写的项目。然而，这并不是100%正确的：苹果不遗余力地使旧的编译代码与新架构兼容，[Roseta 2](https://en.wikipedia.org/wiki/Rosetta_(software))是用来达成这个目标的主要工具.

Docker tried to catch up as well: [the M1 support came out a few weeks ago](https://docs.docker.com/docker-for-mac/apple-silicon/). However, things do not work quite as seamless as:

Docker 也试图迎头赶上 [the M1 support came out a few weeks ago](https://docs.docker.com/docker-for-mac/apple-silicon/).但是，事情并不像预期那样无缝运行：



Docker relies on [qemu](https://www.qemu.org/) to emulate Intel's architecture on M1 chips. Unfortunately, QEMU doesn't work quite well: _However, attempts to run Intel-based containers on Apple Silicon machines can crash as QEMU sometimes fails to run the container. Filesystem change notification APIs (e.g. inotify) do not work under QEMU emulation [...]. Therefore, we recommend that you run ARM64 containers on Apple Silicon machines. These containers are also faster and use less memory than Intel-based containers._ (See docker [documentation for more details](https://docs.docker.com/docker-for-mac/apple-silicon/))

Docker依靠 [qemu](https://www.qemu.org/) 在M1芯片上模拟英特尔的架构。然而事实是 QEMU有时不能很好地工作：_在 Apple Silicon 机器上尝试运行基于 Intel 的容器可能会崩溃，因为 QEMU 有时无法运行容器。例如文件变更 API [File System Events](https://developer.apple.com/documentation/coreservices/file_system_events)（例如 inotify）在 QEMU 仿真下不起作用 [...]。  因此，我们建议您在 Apple Silicon 机器上运行 ARM64 容器。与基于 Intel 的容器相比，这些容器也更快且使用更少的内存_。


How to build an image for `arm64`[#](#object)
---------------------------------------------

Luckily, [Docker has announced](https://www.docker.com/blog/multi-platform-docker-builds/) the support of cross CPU architecture builds a few weeks ago. **buildx** - an experimental feature that allows building images for a certain architecture - arm64, amd64, etc.

幸运的是，[Docker 已经宣布](https://www.docker.com/blog/multi-platform-docker-builds/) 一项实验性功能 **[buildx](https://github.com/docker/buildx)**，它可以为特定架构（arm64、amd64 等）构建映像。

### Step 1. Enable experimental features[#](#step-1-enable-experimental-features)

If you’re using Docker Desktop for Mac you should just enable experimental features on the Docker CLI if you haven’t already – just edit **~/.docker/config.json** to include the following:

如果您使用的是 Mac 版 Docker Desktop，您只需在 Docker CLI 上启用实验性功能 —— 编辑 **~/.docker/config.json** 添加以下内容：
```
{
  ...
  "experimental": "enabled"
}

```

Restart your Docker Desktop after that

之后重新启动您的 Docker Desktop

If you’re using Linux, just run the following command:


如果您使用的是 Linux，只需运行以下命令：

```
docker run --privileged --rm docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64

```

And restart Docker with `service docker restart`

并使用 `service docker restart` 重启 Docker

### Step 2. Create a new builder instance[#](#step-2-create-a-new-builder-instance)

Create a new builder instance:

创建一个新的构建器实例：

```
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

```
docker buildx build --platform linux/amd64,linux/arm64 -t company/image_name .

```

**platform** flag specifies for which platforms Docker image will be built. Docker support 10 platforms, but probably you shouldn use only `linux/amd64` (Intel) and `linux/arm64` (M1):

**platform** 标志指定将为哪些平台构建 Docker 映像。 目前Docker 支持 10 个平台，但可能你应该只使用 `linux/amd64` (Intel) 和 `linux/arm64` (M1)：

![](https://jitsu.com/img/blog/multi-platform-docker-builds/docker-hub-result.png)

Docker Hub result

Reference
--------------------------
- [buildx](https://github.com/docker/buildx)
- [qemu](https://www.qemu.org/)