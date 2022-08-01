> 本文由  [www.makeuseof.com](https://www.makeuseof.com/go-version-118-release-new/)的中文翻译版本，内容有删减。关于 工作区，泛型和模糊测试更详细的说明，请参考 [Go 1.18 的那些事 —— 工作区、模糊测试、泛型](https://mp.weixin.qq.com/s/StSee4HVD7UlprEj-K-bQg)


The Go programming language has evolved a lot since it was first published in 2009. Go 1.18 was a highly-anticipated release due to its support for generics and many other important updates.

自 2009 年首次发布以来，Go 编程语言已经进步了很多。Go 1.18 是一个备受期待的版本，因为它支持泛型和许多其他重要更新。

Go released version 1.18 in March 2022. Here's the low-down on the most significant changes.

Go 于 2022 年 3 月发布了 1.18 版。以下是最重大变化的简要介绍。

## Support for Generics (支持泛型)
--------------------

Generic programming lets you write functions that can accept and return more flexible types.

泛型编程允许您编写在编写函数时，可以用更灵活的类型来定义输入和返回值。

Before support for generics, you needed to explicitly state parameter types and return types. The simplest form of generics allows you to specify untyped parameters:

在支持泛型之前，您需要显式声明参数类型和返回类型。 最简单的泛型形式允许您指定无类型参数：

``` json
func PrintAnything[T any](thing T) {
    fmt.Println(thing)
}
```

But generics offer much more power than just this. You can declare almost any combination and granularity of type for parameters. For example, you can use the **constraints** package to write a function that operates on any value that you can order. This includes int, floats, and strings. Here's an example that demonstrates the syntax:

但是泛型其实功能更加强大，你可以声明任何类型的参数。例如，您可以使用 **constraints** 包来编写一个函数，该函数可以操作任何可以排序的值。包括 int、float 和 string等。这里是一个示例：


``` json
import "golang.org/x/exp/constraints"

 func Max[T constraints.Ordered](input []T) (max T) {
    for _, v := range input {
        if v > max {
            max = v
        }
    }

     return max
}

```

Note that this function uses the generic type, **constraints.Ordered**, to declare its parameter and its return type.


请注意这个函数使用了泛型和**constraints.Ordered** 来声明其参数和返回类型。

Generics provide for uncertainty and flexibility in code. The Generics proposal and changes are backward compatible.

泛型可以给代码添加灵活性和可扩展性。泛型提案和变更是向后兼容的。

## Fuzz Testing （模糊测试）
------------

Fuzzing is a [software testing technique](https://www.makeuseof.com/regression-testing-vs-unit-testing/) that validates a program with erroneous, unexpected, or unpredictable data.

模糊测试是一种 [软件测试技术](https://www.makeuseof.com/regression-testing-vs-unit-testing/)，它用于验证一个程序的错误、不可预期或不可预测的数据。

The **testing** package introduces fuzzing in 1.18, so to define a fuzz, you have to import it from the standard library:

**testing** 包在 1.18 中引入了模糊测试，因此要定义模糊测试，您必须从标准库中导入它：

```json
import "testing"
```

After importing the **testing** package, you can pass an identifier of type ***testing.F** to the test function.

在导入 **testing** 包后，您可以将一个 ***testing.F** 类型的标识符传递给测试函数。


``` json
func testFunc(f *testing.F) {
    
}

```

Fuzzing generates input parameters for testing code. The outcome of fuzzing is unpredictable since the inputs are not user-defined. Fuzzing should help you write better code tests and catch bugs you didn’t know existed.

模糊测试会自动生成测试代码的输入参数。 模糊测试的结果是不可预测的，因为输入不是用户定义的。 模糊测试应该帮助您写更好的代码测试，并捕获您没有知道存在的 bug。

## Go Workspace Support (工作区的支持)
--------------------

Workspaces are directories of similar source code that make up a project or a bigger unit. Workspaces make it easier to manage and debug code by grouping similar code based on functionality.

工作区是一个相似的源代码的目录，它们组成一个项目或者一个大的单元。 工作区可以使您更容易地管理和调试代码，通过将相似的代码组合在一起来更好地管理代码。

Conventionally, you subdivide Go projects into source code (**src**) and executable files (**bin**). The Go toolchain builds source code from the former into executables in the latter. Go workspaces allow developers to work with multiple workspaces using Go modules with a particular main module.

按照惯例，您可以将 Go 项目分成源代码 (**src**) 和可执行文件 (**bin**)。 Go 工具链将源代码从前到后编译成可执行文件。 Go 工作区可以使开发者使用 Go 模块来多个工作区。

The command for creating workspaces is:

创建工作区的命令是：

```json
$ go work <command>
```

Use the **work** command with subcommands like:

*   **init** → creates a workspace in the specified directory.
*   **use** → adds a new module to go.work, the go workspace file.
*   **edit** → edits the go workspace file.
*   **sync** → synchronizes dependencies from the build list to the workspace modules.

可以用 **work** 搭配以下子参数使用，例如：
* **init** → 在指定目录中创建工作区。
* **use** → 向 go.work 添加一个新模块，即 go 工作区文件。
* **edit** → 编辑 go 工作区文件。
* **sync** → 将构建列表中的依赖项同步到工作区模块。


The inclusion of workspaces to go would increase productivity as planned in developing the language.

按照开发语言的计划，在项目中使用工作区将提高生产力。

## Performance Improvements （性能提升）
------------------------

[Go](http://go.dev/) version 1.18 now supports ARM64 Apple M1 and 64-bit PowerPC in the API calling convention. This results in over 10% gain in CPU performance for users of these devices.

[Go](http://go.dev/) 1.18 版在 API 调用约定中支持 ARM64 Apple M1 和 64 位 PowerPC。 这为这些设备的用户带来了超过 10% 的 CPU 性能提升。

Declared and unused variables in functions are now logged as errors on program compilation.

在函数中声明但未使用的变量会在编译时记录为错误。


The **go build** command, along with other related commands, supports the **-asan** flag, which supports interoperability with C/C++. This will help Go developers use Go programs with C and C++ programs.

**go build** 命令，以及其他相关命令，支持 **-asan** 参数，这将帮助 Go 的开发者将 Go 程序与 C 和 C++ 程序配合使用。

## Other Important Updates （其他重要的更新）
-----------------------

The **go get** command no longer installs packages in module-aware mode, which is a big change from when you [first got started with Go](https://www.makeuseof.com/go-getting-started/). The **go install** command replaces **get** to adjust module dependencies across workspaces.

**go get** 命令不再在模块化模式下安装包，这是从 [first got started with Go](https://www.makeuseof.com/go-getting-started/) 以来的一个大的变化。 **go install** 命令取代 **get** 来调整工作区间的模块依赖关系。

Since the type checker now handles generics, error messages may log differently than in previous versions.

由于类型检查器现在也处理泛型的检查，导致错误消息可能会与之前的版本不同。

Time taken for programs to compile may be slower in 1.18. But this will not affect execution time once Go has compiled your program.

在 1.18 中编译程序所花费的时间可能会更慢。 但这不会影响 Go 编译程序后的执行时间。

You can find the full details of all the latest changes in the release notes for [Go 1.18](https://tip.golang.org/doc/go1.18).

你可以在 [Go 1.18](https://tip.golang.org/doc/go1.18) 的发行说明中找到关于最新更新的详细信息。


## 扩展阅读
* [Go 1.18 的那些事 —— 工作区、模糊测试、泛型](https://mp.weixin.qq.com/s/StSee4HVD7UlprEj-K-bQg)
* [An Introduction To Generics](https://go.dev/blog/intro-generics)