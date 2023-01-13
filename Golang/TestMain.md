> [Why use TestMain for testing in Go?
](https://medium.com/goingogo/why-use-testmain-for-testing-in-go-dafb52b406bc)  的中文翻译版本。


TestMain 是在跟随 Go 1.4 一起发布的功能，本文将对其进行讲解。

### 什么是测试主函数 (What is TestMain function exactly)
--------------------------------------


首先，它并不是用来测试你的程序的`Main`函数。和之前的Golang版本相比，TestMain 函数整体上为测试提供了更多控制。

```Go
func TestMain(m *testing.M)
```

如果测试代码包含一个TestMain函数，那么该函数将被其他函数调用而不是直接运行。Struct M 包含了访问和运行测试的方法。但是当使用TestMain时，需要注意以下几点：
1.  您可以在测试文件中定义此函数： `func TestMain(m *testing.M)`.
2.  由于在给定包中函数的名称必须是唯一，因此您只能在每个包中定义一次`TestMain`。如果您的包下面有多个测试文件，必须为 `TestMain` 函数选择一个合适的位置.
3.  Struct `testing.M` 有一个名为 `Run()` 的函数，它将运行包中的所有测试案例， 同时`Run()` 返回一个可以传递给 `os.Exit` 的退出代码.
4.  `func TestMain(m *testing.M)`一个最简单的实现像这样: `func TestMain(m *testing.M) { os.Exit(m.Run()) }`.
5.  如果在return的时候不调用 `os.Exit`，测试结果将返回 0（零），在测试失败也是如此。

示例代码
-------------------

```Go
package mypackagename

import (
    "testing"
    "os"
)

func TestMain(m *testing.M) {
    log.Println("Do stuff BEFORE the tests!")
    exitVal := m.Run()
    log.Println("Do stuff AFTER the tests!")

    os.Exit(exitVal)
}

func TestA(t *testing.T) {
    log.Println("TestA running")
}

func TestB(t *testing.T) {
    log.Println("TestB running")
}
```

**输出:**

```Shell
2017/12/29 00:32:17 Do stuff BEFORE the tests!
2017/12/29 00:32:17 TestA running
2017/12/29 00:32:17 TestB running
PASS
2017/12/29 00:32:17 Do stuff AFTER the tests!
```


因此，当您需要为测试进行一些全局设置/拆卸时，这可能会派上用场。但请记住，TestMain 只会运行一次，因此如果你需要在每个测试结束后清除数据库或清理 tmp 文件夹,需要自己做。


如前所述，`TestMain`允许在测试运行之前和测试结束后运行相应的代码，通过在测试文件中定义 `init()` 函数也可以实现相应的功能，但是需要在所有测试完成后依次将相应的资源释放。

### 一个更复杂的例子(How about we do a more complex example then that?) 
-------------------------------------------------
假设我们需要做一些数据库相关的测试，我们必须在每次测试之前进行相应的配置，在测试运行之后进行资源的释放。在不使用 TestMain 或任何其他框架的情况下的操作如下：


```Go
func TestDBFeatureA(t *testing.T) {
    models.TestDBManager.Setup()
    defer models.TestDBManager.Exit()

    // Do the tests
}

func TestDBFeatureB(t *testing.T) {
    models.TestDBManager.Setup()
    defer models.TestDBManager.Exit()

    // Do the tests
}
```

正如在上面的测试中看到的那样，必须在每一个测试案例中进行相同的Setup和Exit，并且随着测试suite的叠加，由于在每一个测试用例之前都需要上面的过程，运行测试所需的时间会线性增长。

而使用了 `TestMain()`，代码变得更加简洁了，如下所示：

```Go
func TestDBFeatureA(t *testing.T) {
    defer models.TestDBManager.Reset()

    // Do the tests
}

func TestDBFeatureB(t *testing.T) {
    defer models.TestDBManager.Reset()

    // Do the tests
}

func TestMain(m *testing.M) {
    models.TestDBManager.Setup()
    // os.Exit() does not respect defer statements
    code := m.Run()
    models.TestDBManager.Exit()
    os.Exit(code)
}
```

虽然每个测试仍然必须自行清理，但这只涉及恢复初始数据，这比走数据库迁移更快。

这种方法还减少了代码的冗余，因为我们只有一行代码来管理数据库，而不是两行，这样运行效率会很高。

`TestMain` 函数是一个简单的解决方案，它带来了很多好处。当然，并不是所有的测试都需要`TestMain`，但对于那些需要的人来说，这是一个很好的工具。特别是因为我觉得很多人使用第三方测试框架只是因为它们的设置和拆除功能，特别是那些用于处理许多其他语言的第三方框架的功能。我认为 Go 标准库提供的功能比人们通常意识到的要多，他们可以避免在没有实际需要的情况下引入第三方框架。对于测试来说，这是一个很好的工具。
