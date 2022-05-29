> [Why use TestMain for testing in Go?
](https://medium.com/goingogo/why-use-testmain-for-testing-in-go-dafb52b406bc)的中文翻译版本。


TestMain 是在跟随 Go 1.4 一起发布的功能，本文将对其进行讲解。

**TestMain 函数到底是什么？**
--------------------------------------

No, it is not to test the main function on your program. Basically the TestMain function provides more control over running tests than was available in the prior releases of Go. So if the test code contains a function:

首先，它并不是用来测试你的程序的`Main`函数。和之前的Golang版本相比，TestMain 函数整体上为测试提供了对的更多可控制性。

```
func TestMain(m *testing.M)
```
that function will be called instead of running the tests directly. The M struct contains methods to access and run the tests. But there are some rules to remember when using TestMain function:


1.  You can define this function in your test file: `func TestMain(m *testing.M)`.
2.  Since function names need to be unique in a given package, you can only define the `TestMain` once for each package. If your package has multiple test files, choose a logic place for your single `TestMain` function.
3.  The `testing.M` struct has a single defined function named `Run()`. As you can imagine, it runs all the tests within the package. `Run()` returns an exit code that can be passed to `os.Exit`.
4.  A minimal implementation looks like this: `func TestMain(m *testing.M) { os.Exit(m.Run()) }`.
5.  If you don't call `os.Exit` with the return code, your test command will return 0(zero). Yes, even if a test fails.

如果测试代码包含一个TestMain函数，那么该函数将被其他函数调用而不是直接运行。Struct M 包含了访问和运行测试的方法。但是当使用TestMain试，需要注意一下几点
1.  您可以在测试文件中定义此函数： `func TestMain(m *testing.M)`.
1.  由于在给定包中函数的名称必须是唯一，因此您只能在每个包中定义一次`TestMain`。如果您的包下面有多个测试文件，必须为 `TestMain` 函数选择一个合适的位置.
1.  Struct `testing.M` 有一个名为 `Run()` 的函数，它将运行包中的所有测试案例， 同时`Run()` 返回一个可以传递给 `os.Exit` 的退出代码.
1.  `func TestMain(m *testing.M)`一个最简单的实现像这样: `func TestMain(m *testing.M) { os.Exit(m.Run()) }`.
1.  如果在return的时候不调用 `os.Exit`，测试结果将返回 0（零），在测试失败也是如此。

例子:
-------------------

```
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

```
2017/12/29 00:32:17 Do stuff BEFORE the tests!
2017/12/29 00:32:17 TestA running
2017/12/29 00:32:17 TestB running
PASS
2017/12/29 00:32:17 Do stuff AFTER the tests!
```

So this may come in handy when you need to do some global set-up/tear-down for your tests, but remember, TestMain will only run once, so if you need to clear a database or a tmp folder every each test you need to do it yourself.

因此，当您需要为测试进行一些全局设置/拆卸时，这可能会派上用场。但请记住，TestMain 只会运行一次，因此如果你需要在每个测试结束后清除数据库或清理 tmp 文件夹,需要自己做。

As already mentioned, adding a `TestMain` to a package allows it to run arbitrary code before and after tests run. But only the second part is really new. Running global test setup has always been possible by defining an `init()` function in a test file. The challenge has been aligning that with corresponding shutdown code when all tests have completed.

如前所述，`TestMain`允许在测试运行之前和测试结束后运行相应的代码。但其实这些也可以通过其他的方式实现，通过在测试文件中定义 `init()` 函数也可以实现相应的功能，但是需要在所有测试完成后将其与相应的资源的释放。

How about we do a more complex example then that?
-------------------------------------------------

Suppose we need to do some database testing where we had to do some setup before each test and after everything drop what I did and restore the original one. Without using TestMain or any other framework we had to do something like this:

假设我们需要做一些数据库相关的测试，我们必须在每次测试之前进行相应的配置，在测试运行之后进行资源的释放。在不使用 TestMain 或任何其他框架的情况下的操作如下：


```
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

As you can see in every single test which utilized the DB had to go through the same process of setup and exiting and as the testing suite grow, the time it took to run tests grew linearly.

正如在每个使用数据库的测试中看到的那样，必须在每一个测试案例中进行相同的设置和退出过程，并且随着测试套件的叠加，因为在每一个测试用例之前都需要上面的过程，运行测试所需的时间会线性增长。

Now it comes `TestMain()` to the rescue making it possible to run these migrations only once. So now the code would look more like this:

而使用了 `TestMain()`，代码变得更加简洁了，如下所示：
```
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

While each test must still clean up after itself, that would only involve restoring the initial data, which is way faster than doing the schema migrations.

尽管每个测试用例都必须在自己的测试结束后进行资源的释放，但是这只需要还原初始的数据，这比走数据库迁移更快。

This approach also reduces code duplication since we now only have one line for database management in each test instead of two and would run a lot faster without a doubt.

这种方法还减少了代码的冗余，因为我们只有一行代码来管理数据库，而不是两行，这样运行效率会很高。

The TestMain function is a simple solution that it brings a lot to the table. Off course that not all packages will need to implement the TestMain but it is a nice addition for those that need. Specially because I feel like a lot of people use third party testing frameworks just because of their setup and tear down features, specially those used to deal with a lot of third party frameworks in other languages. I just think that the Go Standard Library has a more to offer then usually people realize and sometimes they bring third party framework without the actual need to. Not only related to testing.

`TestMain` 函数是一个简单的解决方案，它带来了很多好处。当然，并不是所有的测试都需要`TestMain`，但对于那些需要的人来说，这是一个很好的工具。特别是因为我觉得很多人使用第三方测试框架只是因为它们的设置和拆除功能，特别是那些用于处理许多其他语言的第三方框架的功能。我认为 Go 标准库提供的功能比人们通常意识到的要多，他们可以避免在没有实际需要的情况下引入第三方框架。对于测试来说，这是一个很好的工具。
