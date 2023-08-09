> 本文是 [Getting started with Go plugin package](https://echorand.me/posts/getting-started-with-golang-plugins/)的中文翻译版本，内容有删减


## 介绍
在这篇文章中，我们将介绍如何在Golang中使用[plugin](https://golang.org/pkg/plugin/)。我们将编写一个名为`driver`的程序，它会加载两个插件并执行它们中共有的某个函数。这个`driver`程序会向第一个插件传递一个整数，第一个插件会对它进行处理。第一个插件的结果会传递给第二个插件，最后`driver`程序将打印出结果。

## 配置
首先我们创建一个名为`golang-plugin-demo`的目录，然后进入该目录，然后创建名为types的文件夹 📁：

```shell
$ mkdir golang-plugin-demo
$ cd $_
```

## 编写一个共享的包
```shell
$ mkdir types
$ cd types/
```


接下来创建`type.go`文件，内容如下：

```go

package types

type InData struct {
        V int
}

type OutData struct {
        V int
}
```

## 编写插件：
返回上一级目录，并创建一个名为`plugin1`的目录：

```shell
$ mkdir plugin1
$ cd plugin1
```

然后创建一个名为`plugin.go`的文件，内容如下：

```go
package main

import "../types"

var Input types.InData
var Output types.OutData
var Name string

func init() {
        Name = "plugin1"
}

func process() types.OutData {
        o := types.OutData{V: Input.V * 2}
        return o
}
func F() {
        Output = process()
}
```

使用以下命令构建插件：

``` shell
$ go build -buildmode=plugin
```

返回上一级目录，并创建一个名为`plugin2`的目录：

```shell
$ mkdir plugin2
$ cd plugin2
```


创建一个名为`plugin.go`的文件，内容如下：

```go
package main

import "../types"

var Input types.InData
var Output types.OutData
var Name string

func init() {
        Name = "plugin2"
}
func process() types.OutData {
        o := types.OutData{V: Input.V * 20}
        return o
}
func F() {
        Output = process()
}
```
使用以下命令构建第二个插件：
```shell
$ go build -buildmode=plugin
```
## 编写 driver 程序
接下来，我们在上一级目录创建一个名为`main.go`的文件，内容如下：

```go
package main

import (
        "log"
        "plugin"
        "./types"
)

func LoadPlugins(plugins []string) {

        d := types.InData{V: 1}
        log.Printf("Invoking pipeline with data: %#v\n", d)
        o := types.OutData{}
        for _, p := range plugins {
                p, err := plugin.Open(p)
                if err != nil {
                        log.Fatal(err)
                }
                pName, err := p.Lookup("Name")
                if err != nil {
                        panic(err)
                }
                log.Printf("Invoking plugin: %s\n", *pName.(*string))

                input, err := p.Lookup("Input")
                if err != nil {
                        panic(err)
                }
                f, err := p.Lookup("F")
                if err != nil {
                        panic(err)
                }

                *input.(*types.InData) = d
                f.(func())()

                output, err := p.Lookup("Output")
                if err != nil {
                        panic(err)
                }
                // Feed the output to the next plugin's input
                d = types.InData{V: output.(*types.OutData).V}
                *input.(*types.InData) = d

                o = *output.(*types.OutData)
        }
        log.Printf("Final result: %#v\n", o)
}

func main() {
        plugins := []string{"plugin1/plugin1.so", "plugin2/plugin2.so"}
        LoadPlugins(plugins)
}
```


目前为止，我们的目录看起来像这样：

```shell
.
├── main.go
├── plugin1
│   ├── plugin1.so
│   └── plugin.go
├── plugin2
│   ├── plugin2.so
│   └── plugin.go
└── types
    └── type.go
```

现在，我们来build并运行我们的driver程序：

```shell
$ go build
$ ./golang-plugin-demo 
2020/05/26 15:49:48 Invoking pipeline with data: types.InData{V:1}
2020/05/26 15:49:48 Invoking plugin: plugin1
2020/05/26 15:49:48 Invoking plugin: plugin2
2020/05/26 15:49:48 Final result: types.OutData{V:40}
```
## 总结

其实用`plugin`包在Golang中实现插件的想法似乎相当简单。编写插件，导出一定的符号（仅限函数和变量），然后在驱动程序中使用它们。插件必须位于`main`包中。你无法从插件中的驱动程序中访问任何类型信息。因此，为了进行任何类型推断（这是必要的），我们可以使用一个类型共享包（就像上面的`InData`和`OutData`一样）。似乎没有一种方法可以将数据从插件“返回”给驱动程序。因此，我们使用插件符号查找来检索插件的“输出”。


*   [Tyk](https://tyk.io/docs/plugins/golang-plugins/golang-plugins/) can be configured by writing Golang plugins.
*   [Gosh](https://github.com/vladimirvivien/gosh) is a shell written in a way where you can write your own commands by making use of Golang plugins.
*   [Discussion on Reddit](https://www.reddit.com/r/golang/comments/b6h8qq/is_anyone_actually_using_go_plugins/) about what folks are using plugins for