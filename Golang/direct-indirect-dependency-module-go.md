> [Direct vs Indirect Dependencies in go.mod file in Go](https://golangbyexample.com/direct-indirect-dependency-module-go/) 的中文翻译版本。




Module 是 Go的依赖工具。根据定义，Module是一组以 **go.mod** 为根目录的相关包的集合。 **go.mod** 文件定义了
* Module 导入路径。
* 用于成功构建项目的依赖项要求。它不仅定义了模块的依赖，同时也确定了依赖的版本。


Module中的依赖可以是两种类型：
*   **Direct**  在项目文件中被直接引用的直接依赖。
*   **Indirect** 
    * 是module的直接依赖的二次依赖。
    * 在go.mod文件中被声明，但是没有被文件直接引用的依赖也被视为indirect依赖。

**go.mod** 文件理论上只记录了direct dependency。然而在下面的情况，它也会记录indirect dependency

* 不在go.mod文件中出现的direct dependency，或者direct dependency项目中不包含go.mod文件。那么这个依赖将被添加到go.mod文件中，并且使用//indirect作为后缀
* 没有被任何文件引用的依赖。

**go.sum** 将记录同时direct和indirect dependencies的校验和

接下来，我们来看一个direct dependency的例子。

```Shell
git mod init learn
```

创建一个文件 **learn.go**

```Golang
package main

import (
	"github.com/pborman/uuid"
)

func main() {
	_ = uuid.NewRandom()
}
```

请注意，我们在**learn.go**中用以下方式指定了依赖

```Golang
"github.com/pborman/uuid"
```

所以 **[github.com/pborman/uuid](http://github.com/pborman/uuid)** 是一个我们的learn 项目的direct dependency，因为它是直接引用到**learn.go**中的。 现在让我们运行以下命令

```
go mod tidy
```

此命令将下载源文件中所需的所有依赖项。 运行此命令后，让我们再次检查 **go.mod** 文件的内容

```Shell
# go.mod
module learn

go 1.14

require github.com/pborman/uuid v1.2.1
```

它列出了在 **learn.go** 文件中指定的direct dependency以及dependency的确切版本。 现在让我们检查一下 go.sum 文件.

```Shell
# go.sum

github.com/google/uuid v1.0.0 h1:b4Gk+7WdP/d3HZH8EJsZpvV7EtDOgaZLtnaNGIu1adA=
github.com/google/uuid v1.0.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
github.com/pborman/uuid v1.2.1 h1:+ZZIw58t/ozdjRaXh/3awHfmWRbzYxJoAdNJxe/3pvw=
github.com/pborman/uuid v1.2.1/go.mod h1:X/NO0urCmaxf9VXbdlT7C2Yzkj2IKimNn4k+gtPdI/k=
```


**go.sum** 文件列出了模块所需的direct and indirect dependency的校验和。 [github.com/google/uuid](http://github.com/google/uuid) 被 [github.com/pborman/uuid](http://github.com/pborman/uuid) 内部使用，因此它作为indirect dependency被记录在 **go.sum** 文件中。

以上我们通过在源文件中添加了一个依赖项，并使用 **go mod tidy** 命令下载该依赖项并将其添加到 **go.mod** 文件中。


让我们通过一个例子来理解它。 为此，让我们首先创建一个模块

```Shell
git mod init learn
```

现在创建一个文件 **learn.go**
```Golang
package main

import (
	"github.com/gocolly/colly"
)

func main() {
	_ = colly.NewCollector()
```

请注意，我们已将 **learn.go** 中的依赖项指定为

```Golang
github.com/gocolly/colly
```

所以 github.com/gocolly/colly 是 learn 项目的direct dependency，因为它是直接导入到模块中的。让我们在 go.mod 文件中添加 colly v1.2.0 版本作为依赖

```Shell
module learn

go 1.14

require	github.com/gocolly/colly v1.2.0
```

Now let’s run the below command

然后运行以下命令

```Shell
go mod tidy
```


运行此命令后，让我们再次检查 **go.mod** 文件的内容。 由于 colly v1.2.0 版本没有 go.mod 文件，所以 colly 需要的所有依赖都将添加到 **go.mod** 文件中，并以 **//indirect** 作为后缀

```Golang
# go.mod
module learn

go 1.14

require (
	github.com/PuerkitoBio/goquery v1.6.0 // indirect
	github.com/antchfx/htmlquery v1.2.3 // indirect
	github.com/antchfx/xmlquery v1.3.3 // indirect
	github.com/gobwas/glob v0.2.3 // indirect
	github.com/gocolly/colly v1.2.0
	github.com/kennygrant/sanitize v1.2.4 // indirect
	github.com/saintfish/chardet v0.0.0-20120816061221-3af4cd4741ca // indirect
	github.com/temoto/robotstxt v1.1.1 // indirect
	golang.org/x/net v0.0.0-20201027133719-8eef5233e2a1 // indirect
	google.golang.org/appengine v1.6.7 // indirect
)
```

所有colly的其他依赖项都以 **//indirect** 为后缀。 所有direct 和 indirect dependencies的校验和也将记录在 go.sum 文件中。


我们还提到，**任何未在任何源文件中导入的依赖项都将被标记为 //indirect**。 为了验证这个说法，让我们删除上面创建的**learn.go**文件，同时清理 **go.mod** 文件以中相关行。 最后运行以下命令

```Go
go get github.com/pborman/uuid
```
现在检查 **go.mod** 文件的内容

```Go
module learn

go 1.14

require github.com/pborman/uuid v1.2.1 // indirect
```


请注意，[github.com/pborman/uuid](github.com/pborman/uuid)被记录为 //indirect，因为没有任何文件中直接引用它。