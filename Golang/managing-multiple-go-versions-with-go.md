> 本文是 [Managing Multiple Go Versions with Go] (https://lakefs.io/blog/managing-multiple-go-versions-with-go/)的中文翻译版本，内容有删减


作为 Go 编程语言的用户，我发现在一个项目中启用运行多个版本非常有用。如果你已经尝试过或考虑过这个功能，那太好了！在本文中，我将介绍启用多个 Go 版本的**时间**和**方法**。最后，我们将讨论为什么这种方法如此强大。


## 什么时候需要多版本的Go (When Do We Need Multiple Go Versions?)
默认情况下，安装 Go 只意味着你可以运行一个 go 命令来构建和测试你的项目。这对于入门很简单，但也可能有局限性。

更灵活的设置是通过 go1.17 或 go1.18 命令启用在同一环境中运行多个版本。另一种替代方法是设置终端的 PATH 环境变量以指向特定的 Go 版本的 SDK。

以下是我发现可能会有有多个go版本的几种情况：

- **项目需求不同** —— 在多个项目之间切换时，通常需要为每个项目使用不同的 Go 版本。
- **创建特定的测试环境** —— 在测试向后兼容性或确保修复漏洞的成功时，重要的是控制运行时版本。
- **保持最前沿** —— 当测试最新 Go 发布中仅可用的新功能或包的行为时

### 必备条件(Prerequisites)

本指南假定您已经知道如何使用 Go 构建和运行程序。具体来说，这意味着您已经安装了 Go 和 Git，并且它们在您的路径中可用。

1.  Go – [https://golang.org/doc/install](https://golang.org/doc/install)
2.  Git – [https://git-scm.com/](https://git-scm.com/)
3.  Modules – [Using Go Modules](https://blog.golang.org/using-go-modules)

## 如何使用多个 Go 版本(How to Work With Multiple Go Versions)
-------------------------------------
我们可以使用 `go install` 命令来下载并安装单个 Go 版本。

运行 `go install golang.org/dl/go<version>@latest` 命令将下载并安装特定 Go 版本的包装 Go 命令。

请注意，如果您运行的 Go 版本低于 Go 1.16，请使用 `go get <package>` 命令替代所有以下示例中的 `go install <package>@latest` 命令。

通过使用 Go 包装器，我们可以下载特定版本的 Go，并为此版本运行 Go 工具链。

 Go v1.18的例子

```bash
$ go install golang.org/dl/go1.18@latest
$ go1.18 download

```
使用包装器 `go1.18`，我们可以使用 Go v1.18 进行构建和测试。例如：

```bash
$ go1.18 mod init hello
go: creating new go.mod: module hello
$ echo 'package main; import "fmt"; func main() { fmt.Println("Hello, World") }' > hello.go
$ go1.18 build
$ ./hello
Hello, World

```

另一个选项是通过设置 `GOROOT` 并将其添加到我们的路径中来使用刚刚下载的版本：

```
$ export GOROOT=$(go1.18 env GOROOT)
$ export PATH=${GOROOT}/bin;${PATH}

```

默认的下载目录是您的 `<home>/sdk` 文件夹。要删除特定版本，只需删除特定版本目录即可。

还有一个特殊标记的版本，称为 `gotip`，代表来自开发者的最新版本的 Go：


```bash
$ go install golang.org/dl/gotip@latest
$ gotip download

```
请注意，gotip 下载并构建当前的开发版本。它还可以接受其他参数 - 分支名称或变更列表 (CL)，并使用它来获取特定的 Go 版本。

## 内部原理 (How It Works Under the Hood)

[https://go.googlesource.com/dl](https://go.googlesource.com/dl) 实现了这个功能。它包括：

- 每个 Go 版本的小型应用程序 - 例如：[`go1.18/main.go`](https://go.googlesource.com/dl/+/refs/heads/master/go1.18/main.go)
- 实现 Go 包装器功能的内部包：[`internal/version/version.go`](https://go.googlesource.com/dl/+/refs/heads/master/internal/version/version.go)
- 一个辅助应用程序，用于生成 Go 包装器代码：[`internal/genv/main.go`](https://go.googlesource.com/dl/+/refs/heads/master/internal/genv/main.go)


对于每个Go版本的 `main.go`,Go的包装器是

```Go
package main

import "golang.org/dl/internal/version"

func main() {
    version.Run("go1.18")
}

```

正如你所看到的，这里没有太多的操作。我们只需调用要提供的版本的公共代码

A packaged version’s `Run` function performs the two main tasks (based on command line arguments):

`Run` 函数执行两个主要任务（基于命令行参数）：

1.  **Download** - `go<version> download`
2.  **Wrapper** - `go<version> <anything else>`

### 下载（Download）

显式指定版本（例如 go1.18）会从 [https://dl.google.com/go](https://dl.google.com/go) 下载特定版本、操作系统和平台的 Go SDK 压缩包，并解压到你的 `<home folder>/sdk` 文件夹下。

接下来，它会通过从下载服务器获取 checksum 文件，使用 sha256 验证压缩包的完整性。最后，它会解压压缩包并创建一个名为 .`.unpacked-success` 的空标记文件。

Gotip 版本 - 当使用 `gotip download` 时，我们只有最新的或特定分支。
```bash
gotip: usage: gotip download [CL number | branch name]
```


`Gotip download` 命令将使用 git 从源码仓库获取源代码文件，然后执行清理操作并运行相关的构建脚本以构建 Go。

与解压特定的发行版不同，我们没有标记文件来标记解压成功。相反，`gotip` 始终尝试获取任何更新并运行构建过程。

### 包装器（Wrapper）

包装功能首先通过验证相关的Go文件夹下是否有有效的标记文件（`.unpacked-success`）来验证我们是否安装了有效的SDK。

如果标记文件缺失，将会给出错误提示信息，指导你运行一个下载请求来进行安装，如：`go<version>: not downloaded. Run 'go<version> download' to install to <home>/sdk/go<version>`。下面这段代码（来自 [internal/version/version.go](https://go.googlesource.com/dl/+/refs/heads/master/internal/version/version.go#57)）格式化了要执行的命令，基于我们要执行的 go 版本，渲染了带有正确路径的环境变量 GOROOT（并将其添加到 PATH 中）。


```go
func runGo(root string) {
    gobin := filepath.Join(root, "bin", "go"+exe())
    cmd := exec.Command(gobin, os.Args[1:]...)
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    newPath := filepath.Join(root, "bin")
    if p := os.Getenv("PATH"); p != "" {
        newPath += string(filepath.ListSeparator) + p
    }
    cmd.Env = dedupEnv(caseInsensitiveEnv, append(os.Environ(), "GOROOT="+root, "PATH="+newPath))
   
    handleSignals()
   
    if err := cmd.Run(); err != nil {
        os.Exit(1)
    }
    os.Exit(0)
}

```

为了为每个版本创建一个Go包装器主程序，使用了一个帮助命令——[genv](https://go.googlesource.com/dl/+/refs/heads/master/internal/genv/main.go)。

它接受我们想要构建Go包装器的go版本作为输入。

首先，它运行`go list -m -json`并解析输出：

```
$ go list -m -json
{
    "Path": "golang.org/dl",
    "Main": true,
    "Dir": "<workspace>/dl",
    "GoMod": "<workspace>/dl/go.mod",
    "GoVersion": "1.11"
}

```

然后它将`Path`与`golang.org/dl`进行匹配，并使用Dir作为包装器代码的目标目录(`<version>/main.go`)。

最后，它使用可用的Go模板来渲染上面所述的Go包装器代码。

## 总结（Summary ）
-------

运行多个 Go 版本的 Go 解决方案非常简单而优雅——它使用与 Go 中的其他工具相同的工具，并提供一种以最小化的编码方式访问不同版本的 Go SDK 的方法。
