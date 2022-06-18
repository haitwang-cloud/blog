> [Direct vs Indirect Dependencies in go.mod file in Go](https://golangbyexample.com/direct-indirect-dependency-module-go/) 的中文翻译版本。


Module is Go support for dependency management. A module by definition is a collection of related packages with **go.mod** at its root.  The **go.mod** file defines the


*   Module import path.

*   Dependencies requirements of the module for a successful build. It defines both module’s dependencies requirement and also lock them to their correct version.

Module 是 Go的依赖工具。根据定义，Module是一组以 **go.mod** 为根目录的相关包的集合。 **go.mod** 文件定义了
* Module 导入路径。
* 用于成功构建项目的依赖项要求。它不仅定义了模块的依赖，同时也确定了依赖的版本。

A dependency of a module can be of two kinds

*   **Direct** -A direct dependency is a dependency which the module directly imports.
*   **Indirect** – It is the dependency that is imported by the module’s direct dependencies. Also, any dependency that is mentioned in the go.mod file but not imported in any of the source files of the module is also treated as an indirect dependency.

module中的依赖可以是两种类型：
*   **Direct**  在项目文件中被直接引用的直接依赖。
*   **Indirect** 
    * 是module的直接依赖的二次依赖。
    * 在go.mod文件中被声明，但是没有被文件直接引用的依赖也被视为indirect依赖。


**go.mod** file only records the direct dependency. However, it may record an indirect dependency in the below cases


*   Any indirect dependency which is not listed in the go.mod file of your direct dependency or if direct dependency doesn’t have a **go.mod** file, then that dependency will be added to the go.mod file with //indirect as the suffix
*   Any dependency which is not imported in any of the source files of the module.

**go.mod** 文件理论上只记录了direct dependency。然而在下面的情况，它也会记录indirect dependency

* 不在go.mod文件中出现的direct dependency，或者direct dependency项目不包含go.mod文件。那么这个依赖将被添加到go.mod文件中，并且使用//indirect作为后缀
* 没有被任何文件引用的依赖。

**go.sum** will record the checksum of both direct and indirect dependencies
**go.sum** 将记录同时direct和indirect dependencies的校验和

Let’s see an example of direct dependency.  For that let’s first create a module

接下来，我们来看一个direct dependency的例子。

```
git mod init learn
```

Now create a file **learn.go**

创建一个文件 **learn.go**

```
package main

import (
	"github.com/pborman/uuid"
)

func main() {
	_ = uuid.NewRandom()
}
```

Notice that we have specified the dependency in the **learn.go** as

请注意，我们在**learn.go**中用以下方式指定了依赖

```
"github.com/pborman/uuid"
```

So **[github.com/pborman/uuid](http://github.com/pborman/uuid)** is a direct dependency of the learn module as it is directly imported in the module. Now let’s run the below command

所以 **[github.com/pborman/uuid](http://github.com/pborman/uuid)** 是一个我们的learn 项目的direct dependency，因为它是直接引用到**learn.go**中的。 现在让我们运行以下命令

```
go mod tidy
```

This command will download all the dependencies that are required in your source files. After running this command let’s now let’s again examine the contents of **go.mod** file

此命令将下载源文件中所需的所有依赖项。 运行此命令后，让我们再次检查 **go.mod** 文件的内容

```
# go.mod
module learn

go 1.14

require github.com/pborman/uuid v1.2.1
```

It lists direct dependency which was specified in the **learn.go** file along with exact version of the dependency as well. Now let’s check the go.sum file as well

它列出了在 **learn.go** 文件中指定的direct dependency以及dependency的确切版本。 现在让我们检查一下 go.sum 文件.

```
# go.sum

github.com/google/uuid v1.0.0 h1:b4Gk+7WdP/d3HZH8EJsZpvV7EtDOgaZLtnaNGIu1adA=
github.com/google/uuid v1.0.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
github.com/pborman/uuid v1.2.1 h1:+ZZIw58t/ozdjRaXh/3awHfmWRbzYxJoAdNJxe/3pvw=
github.com/pborman/uuid v1.2.1/go.mod h1:X/NO0urCmaxf9VXbdlT7C2Yzkj2IKimNn4k+gtPdI/k=
```

**go.sum** file lists down the checksum of direct and indirect dependency required by the module.  [github.com/google/uuid](http://github.com/google/uuid) is internally used by the [github.com/pborman/uuid](http://github.com/pborman/uuid) and it is an indirect dependency of the module and hence it is recorded in the **go.sum** file.

**go.sum** 文件列出了模块所需的direct and indirect dependency的校验和。 [github.com/google/uuid](http://github.com/google/uuid) 被 [github.com/pborman/uuid](http://github.com/pborman/uuid) 内部使用，因此它作为indirect dependency被记录在 **go.sum** 文件中。

The above way we added a dependency in the source file and used **go mod tidy** command to download that dependency and add it in the **go.mod** file.

以上我们通过在源文件中添加了一个依赖项，并使用 **go mod tidy** 命令下载该依赖项并将其添加到 **go.mod** 文件中。

Let’s understand it with an example. For that let’s first create a module

让我们通过一个例子来理解它。 为此，让我们首先创建一个模块

```
git mod init learn
```

Now create a file **learn.go**

```
package main

import (
	"github.com/gocolly/colly"
)

func main() {
	_ = colly.NewCollector()
```

Notice that we have specified the dependency in the **learn.go** as

请注意，我们已将 **learn.go** 中的依赖项指定为

```
github.com/gocolly/colly
```

So github.com/gocolly/colly is a direct dependency of the learn module as it is directly imported in the module.Let’s add colly version v1.2.0 as a dependency in the go.mod file

所以 github.com/gocolly/colly 是 learn 项目的direct dependency，因为它是直接导入到模块中的。让我们在 go.mod 文件中添加 colly v1.2.0 版本作为依赖

```
module learn

go 1.14

require	github.com/gocolly/colly v1.2.0
```

Now let’s run the below command

然后运行以下命令

```
go mod tidy
```

After running this command let’s now let’s again examine the contents of **go.mod** file. Since colly version v1.2.0 doesn’t have a go.mod file , all dependencies required by colly will be added to the **go.mod** file with **//indirect** as suffix

运行此命令后，让我们再次检查 **go.mod** 文件的内容。 由于 colly v1.2.0 版本没有 go.mod 文件，所以 colly 需要的所有依赖都将添加到 **go.mod** 文件中，并以 **//indirect** 作为后缀

```
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

All other dependencies are suffixed by **//indirect**. Also checksum of all direct and indirect dependencies will be recorded in the go.sum file.

所有colly的其他依赖项都以 **//indirect** 为后缀。 所有direct 和 indirect dependencies的校验和也将记录在 go.sum 文件中。

We also mentioned that any dependency which is not imported in any of the source file will be marked as //indirect. To illustrate that delete the **learn.go** created above. Also clean up **go.mod** file to remove all require lines. Now run below command

我们还提到，**任何未在任何源文件中导入的依赖项都将被标记为 //indirect**。 为了验证这个说法，让我们删除上面创建的**learn.go**文件，同时清理 **go.mod** 文件以中相关行。 最后运行以下命令

```
go get github.com/pborman/uuid
```

Now check the contents of **go.mod** file
现在检查 **go.mod** 文件的内容

```
module learn

go 1.14

require github.com/pborman/uuid v1.2.1 // indirect
```

Notice that the dependency is recorded as //indirect as it is not imported in any of the source files.

请注意，[github.com/pborman/uuid](github.com/pborman/uuid)被记录为 //indirect，因为没有任何文件中直接引用它。