> 本文是 [How To Upgrade Golang Dependencies](https://golang.cafe/blog/how-to-upgrade-golang-dependencies.html)的中文翻译版本，内容有删减


无论您是否使用Go modules，使用`go get`命令升级Go依赖项都是一个简单的任务。以下是升级Go依赖项的几个示例。


### 如何把依赖升级到最新的版本 

下面的命令将更新您的go.mod和go.sum文件（针对 example.com/pkg）

```bash
go get example.com/pkg
```

### 如何将依赖项及其所有子依赖项升级到最新版本

如果您需要将依赖项及其所有子依赖项升级到最新版本（针对 example.com/pkg）


```bash
go get -u example.com/pkg
```

### 如何查看可用的依赖项升级

为了查看所有直接和间接依赖项的可用次要和补丁升级

```
go list -u -m all
```

### 如何一次性升级所有依赖项

为了一次性升级所有依赖项，运行以下命令

这将升级所有依赖的**最新的**或**次要的**版本


```
go get -u ./...
```


下面这个命令会测试依赖项的升级发现是否有问题

```
go get -t -u ./...
```

### 如何使用Go modules升级到特定版本

使用上面描述的相同机制，我们可以使用`go get`命令升级到特定的依赖项

```bash
get foo@v1.6.2
```

或者指定一个git提交哈希值

```bash
go get foo@e3702bed2
```

或者您可以在[Module Queries](https://golang.org/cmd/go/#hdr-Module_queries)中进一步探索定义的语义

### 测试升级后的依赖项

为了确保您的包在升级后正常工作，您可能需要运行以下命令来测试它们是否正常工作
```bash
go test all
```


如果您想了解有关Go modules的更多信息，请访问官方文档网站[https://github.com/golang/go/wiki/Modules#how-to-upgrade-and-downgrade-dependencies](https://github.com/golang/go/wiki/Modules#how-to-upgrade-and-downgrade-dependencies)
