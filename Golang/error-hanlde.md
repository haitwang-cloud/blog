> 本文是 [Effective Error Handling in Golang](https://earthly.dev/blog/golang-errors/)的中文翻译版本，内容有删减。

其他优秀的Golang error handle 文章：
- [一套优雅的 Go 错误问题解决方案](https://mp.weixin.qq.com/s/RFF2gSikqXiWXIaOxQZsxQ)

Error handling in Go is a little different than other mainstream programming languages like Java, JavaScript, or Python. Go’s built-in errors don’t contain stack traces, nor do they support conventional `try`/`catch` methods to handle them. Instead, errors in Go are just values returned by functions, and they can be treated in much the same way as any other datatype - leading to a surprisingly lightweight and simple design.

Go 中的 Errors 处理与其他主流编程语言（如 Java、JavaScript 或 Python）略有不同。Go 的内置 Errors 不包含堆栈跟踪，也不支持传统的 `try`/`catch` 方法来处理。相反，Go 中的 Errors 只是函数返回的值，它们可以像处理任何其他数据类型一样被处理, 它为 Go 带来了令人惊讶的轻量级和简单的设计。

In this article, I’ll demonstrate the basics of handling errors in Go, as well as some simple strategies you can follow in your code to ensure your program is robust and easy to debug.

在本文中，我将从 Go 中处理错误的基础知识开始讲解，同时还有在代码中可以遵循的一些简单策略，以确保您的程序是健壮的和易于调试的。

## The Error Type[](#the-error-type)

The error type in Go is implemented as the following interface:

Go 中的错误类型是通过以下接口实现的：

```
type error interface {
    Error() string
}

```

So basically, an error is anything that implements the `Error()` method, which returns an error message as a string. It’s that simple!

所以，Go 中的错误是实现了 `Error()`的任何方法，该方法返回一个字符串类型的错误消息。就是这么简单！

### Constructing Errors[](#constructing-errors)

Errors can be constructed on the fly using Go’s built-in `errors` or `fmt` packages. For example, the following function uses the `errors` package to return a new error with a static error message:

Errors 可以使用 Go 的内置 `errors` 或 `fmt` 包进行构造。例如，以下函数使用 errors 包返回一个带有静态错误消息的 error：

```
package main

import "errors"

func DoSomething() error {
    return errors.New("something didn't work")
}

```

Similarly, the `fmt` package can be used to add dynamic data to the error, such as an `int`, `string`, or another `error`. For example:

类似地，`fmt` 包可以用于将数据动态添加到 error 中，例如 `int`、`string` 或另一个 `error`。例如：

```
package main

import "fmt"

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("can't divide '%d' by zero", a)
    }
    return a / b, nil
}

```

Note that `fmt.Errorf` will prove extremely useful when used to wrap another error with the `%w` format verb - but I’ll get into more detail on that further down in the article.

请注意，当使用 `%w` 格式去 wrap 另一个错误时，`fmt.Errorf` 将非常有用（本文的后面进一步详细介绍）。

There are a few other important things to note in the example above.

- Errors can be returned as `nil`, and in fact, it’s the default, or “zero”, value of on error in Go. This is important since checking `if err != nil` is the idiomatic way to determine if an error was encountered (replacing the `try`/`catch` statements you may be familiar with in other programming languages).
- Errors are typically returned as the last argument in a function. Hence in our example above, we return an `int` and an `error`, in that order.
- When we do return an error, the other arguments returned by the function are typically returned as their default “zero” value. A user of a function may expect that if a non-nil error is returned, then the other arguments returned are not relevant.
- Lastly, error messages are usually written in lower-case and don’t end in punctuation. Exceptions can be made though, for example when including a proper noun, a function name that begins with a capital letter, etc.

上面的例子中有一些其他需要注意的事情。

- Errors 可以返回为 `nil`，事实上，它是 Go 中 error 的默认值或“零”值。这很重要，因为检查 `if err != nil` 是惯用的方法来确定是否遇到错误（替换您可能熟悉的其他编程语言中的 `try`/`catch` 语句）。
- Errors 通常作为函数的最后一个参数返回。因此，在上面的示例中，我们按顺序返回一个 `int` 和一个 `error`。
- 当返回一个 Error 时，函数返回的其他参数通常返回为它们的默认`nil`值。函数的调用者可能希望如果返回了非 `nil` 错误，则返回的其他参数不相关。
- 最后，Errors **通常以小写字母开头，不以标点符号结尾。但在某些情况下也存在例外: 例如包含专有名词、以大写字母开头的函数名等**。


### Defining Expected Errors (按照预期定义错误)


Another important technique in Go is defining expected Errors so they can be checked for explicitly in other parts of the code. This becomes useful when you need to execute a different branch of code if a certain kind of error is encountered.

另一个 Go 中的重要点是按照预期定义错误，以便可以在代码的其他部分中显式检查它们。当您在遇到某种错误需要执行不同的代码分支时，这很有用。

#### Defining Sentinel Errors (定义哨兵错误)
Building on the `Divide` function from earlier, we can improve the error signaling by pre-defining a “Sentinel” error. Calling functions can explicitly check for this error using `errors.Is`:

基于前面的 `Divide` 函数，我们可以通过预定义一个“哨兵”错误来改进 Errors 处理, 在其他函数中时可以使用 `errors.Is` 显式检查此错误：

```
package main

import (
    "errors"
    "fmt"
)

var ErrDivideByZero = errors.New("divide by zero")

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, ErrDivideByZero
    }
    return a / b, nil
}

func main() {
    a, b := 10, 0
    result, err := Divide(a, b)
    if err != nil {
        switch {
        case errors.Is(err, ErrDivideByZero):
            fmt.Println("divide by zero error")
        default:
            fmt.Printf("unexpected division error: %s\n", err)
        }
        return
    }

    fmt.Printf("%d / %d = %d\n", a, b, result)
}

```

#### Defining Custom Error Types（自定义 Error 类型）

Many error-handling use cases can be covered using the strategy above, however, there can be times when you might want a little more functionality. Perhaps you want an error to carry additional data fields, or maybe the error’s message should populate itself with dynamic values when it’s printed.

大多数的Errors 处理可以采用上述的策略，然而有时您可能需要更多的功能。比如您希望 Errors 携带其他数据字段，或者用动态值填充 Errors 消息。

You can do that in Go by implementing custom errors type.

你可以通过实现自定义错误类型来实现这一点。

Below is a slight rework of the previous example. Notice the new type `DivisionError`, which implements the `Error` `interface`. We can make use of `errors.As` to check and convert from a standard error to our more specific `DivisionError`.

下面是前面例子的一个小改动。我们用 `DivisionError`实现了 `Error` `interface`。我们可以使用 `errors.As` 来检查并更具体的 `DivisionError`。


```
package main

import (
    "errors"
    "fmt"
)

type DivisionError struct {
    IntA int
    IntB int
    Msg  string
}

func (e *DivisionError) Error() string {
    return e.Msg
}

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, &DivisionError{
            Msg: fmt.Sprintf("cannot divide '%d' by zero", a),
            IntA: a, IntB: b,
        }
    }
    return a / b, nil
}

func main() {
    a, b := 10, 0
    result, err := Divide(a, b)
    if err != nil {
        var divErr *DivisionError
        switch {
        case errors.As(err, &divErr):
            fmt.Printf("%d / %d is not mathematically valid: %s\n",
              divErr.IntA, divErr.IntB, divErr.Error())
        default:
            fmt.Printf("unexpected division error: %s\n", err)
        }
        return
    }

    fmt.Printf("%d / %d = %d\n", a, b, result)
}

```

Note: when necessary, you can also customize the behavior of the `errors.Is` and `errors.As`. See [this Go.dev blog](https://go.dev/blog/go1.13-errors) for an example.

请注意：您还可以在需要时自定义 `errors.Is` 和 `errors.As` 的行为。可以参考 [this Go.dev blog](https://go.dev/blog/go1.13-errors) 获取示例。

Another note: `errors.Is` was added in Go 1.13 and is preferable over checking `err == ...`. More on that below.

另请注意：`errors.Is` 是在 Go 1.13 版本中添加的，它比使用`err == ...` 更适合。下面会介绍更多。

## Wrapping Errors (包装 Errors)

In these examples so far, the errors have been created, returned, and handled with a single function call. In other words, the stack of functions involved in “bubbling” up the error is only a single level deep.

目前为止，这些例子中的 Errors 都是在单个函数调用中创建、返回和处理的。换句话说，处理 Errors 的函数调用栈只有一层。

Often in real-world programs, there can be many more functions involved - from the function where the error is produced, to where it is eventually handled, and any number of additional functions in-between.

然而在实际的程序中，可能会涉及更多的函数 - 从初始产生 Errors 的函数，到最终处理 Errors 的函数，以及介于两者之间的任意数量的附加函数。

In Go 1.13, several new error APIs were introduced, including `errors.Wrap` and `errors.Unwrap`, which are useful in applying additional context to an error as it “bubbles up”, as well as checking for particular error types, regardless of how many times the error has been wrapped.

在 Go 1.13 中，引入了几个新的 Errors API，包括 `errors.Wrap` 和 `errors.Unwrap`，它们在将额外的上下文应用于 Errors 时非常有用，它可以在 Errors 被wrap多次时，达到检查特定的 Errors 类型。


> **A bit of history**: Before Go 1.13 was released in 2019, the standard library didn’t contain many APIs for working with errors - it was basically just `errors.New` and `fmt.Errorf`. As such, you may encounter legacy Go programs in the wild that do not implement some of the newer error APIs. Many legacy programs also used 3rd-party error libraries such as [`pkg/errors`](https://github.com/pkg/errors). Eventually, [a formal proposal](https://go.googlesource.com/proposal/+/master/design/go2draft-error-inspection.md) was documented in 2018, which suggested many of the features we see today in Go 1.13+.

>  **历史趣闻** 在 2019 年发布的 Go 1.13 之前，标准库并没有提供许多用于处理 Errors 的 API - 它基本上只是 `errors.New` 和 `fmt.Errorf`。因此，您可能会遇到在野外使用旧版 Go 程序的情况，这些程序没有实现一些较新的 Errors API。许多旧版程序还使用了第三方 Errors 库，例如 [`pkg/errors`](

### The Old Way (Before Go 1.13) (旧版)

It’s easy to see just how useful the new error APIs are in Go 1.13+ by looking at some examples where the old API was limiting.

通过查看一些旧版 API 的例子，可以很容易地看出 Go 1.13+ 中新的 Errors API 多有用。

Let’s consider a simple program that manages a database of users. In this program, we’ll have a few functions involved in the lifecycle of a database error.

让我们考虑一个简单的程序，它管理着用户的数据库。在这个程序中，我们将有一些函数参与到数据库 Errors 的生命周期中。

For simplicity’s sake, let’s replace what would be a real database with an entirely “fake” database that we import from `"example.com/fake/users/db"`.

为了简单起见，让我们将真实的数据库替换为完全“假的”数据库，我们从 `"example.com/fake/users/db"` 导入它

Let’s also assume that this fake database already contains some functions for finding and updating user records. And that the user records are defined to be a struct that looks something like:

我们还假设这个数据库已经包含了一些用于查找和更新用户记录的函数。并且用户记录被定义为一个结构体：

```
package db

type User struct {
  ID       string
  Username string
  Age      int
}

func FindUser(username string) (*User, error) { /* ... */ }
func SetUserAge(user *User, age int) error { /* ... */ }

```

Here’s our example program:

这是我们的示例程序：


```
package main

import (
    "errors"
    "fmt"

    "example.com/fake/users/db"
)

func FindUser(username string) (*db.User, error) {
    return db.Find(username)
}

func SetUserAge(u *db.User, age int) error {
    return db.SetAge(u, age)
}

func FindAndSetUserAge(username string, age int) error {
  var user *User
  var err error

  user, err = FindUser(username)
  if err != nil {
      return err
  }

  if err = SetUserAge(user, age); err != nil {
      return err
  }

  return nil
}

func main() {
    if err := FindAndSetUserAge("bob@example.com", 21); err != nil {
        fmt.Println("failed finding or updating user: %s", err)
        return
    }

    fmt.Println("successfully updated user's age")
}

```

Now, what happens if one of our database operations fails with some `malformed request` error?

现在，如果我们的数据库操作失败了，会发生什么呢？

The error check in the `main` function should catch that and print something like this:

`main` 函数中的错误检查应该捕获到这个错误，并打印出类似这样的内容：



```
failed finding or updating user: malformed request

```

But which of the two database operations produced the error? Unfortunately, we don’t have enough information in our error log to know if it came from `FindUser` or `SetUserAge`.

但是是哪个数据库操作产生了错误呢？不幸的是，我们在错误日志中没有足够的信息来知道它是来自 `FindUser` 还是 `SetUserAge`。

Go 1.13 adds a simple way to add that information.

Go 1.13 引入了一种简单的方法来添加这些信息。

### Errors Are Better Wrapped (更好地包装Errors)

The snippet below is refactored so that is uses `fmt.Errorf` with a `%w` verb to “wrap” errors as they “bubble up” through the other function calls. This adds the context needed so that it’s possible to deduce which of those database operations failed in the previous example.

下面的代码片段使用 `fmt.Errorf` 和 `%w`进行了重构，通过其他函数层层调用来“wrap”错误 。通过添加调用的上下文信息，可以推断出在上一示例中哪些数据库操作失败。

```
package main

import (
    "errors"
    "fmt"

    "example.com/fake/users/db"
)

func FindUser(username string) (*db.User, error) {
    u, err := db.Find(username)
    if err != nil {
        return nil, fmt.Errorf("FindUser: failed executing db query: %w", err)
    }
    return u, nil
}

func SetUserAge(u *db.User, age int) error {
    if err := db.SetAge(u, age); err != nil {
      return fmt.Errorf("SetUserAge: failed executing db update: %w", err)
    }
}

func FindAndSetUserAge(username string, age int) error {
  var user *User
  var err error

  user, err = FindUser(username)
  if err != nil {
      return fmt.Errorf("FindAndSetUserAge: %w", err)
  }

  if err = SetUserAge(user, age); err != nil {
      return fmt.Errorf("FindAndSetUserAge: %w", err)
  }

  return nil
}

func main() {
    if err := FindAndSetUserAge("bob@example.com", 21); err != nil {
        fmt.Println("failed finding or updating user: %s", err)
        return
    }

    fmt.Println("successfully updated user's age")
}

```

If we re-run the program and encounter the same error, the log should print the following:

如果我们重新运行程序并遇到相同的错误，日志应该打印出以下内容：

```
failed finding or updating user: FindAndSetUserAge: SetUserAge: failed executing db update: malformed request

```

Now our message contains enough information that we can see the problem originated in the `db.SetUserAge` function. Phew! That definitely saved us some time debugging!

现在，我们的Error消息包含了足够的信息，我们可以看到问题是在 `db.SetUserAge` 函数中产生的。唷！这无疑为我们节省了一些调试时间！

If used correctly, error wrapping can provide additional context about the lineage of an error, in ways similar to a traditional stack-trace.

如果使用得当，错误wrap可以以类似于传统堆栈跟踪的方式提供有关错误谱系的附加上下文。

Wrapping also preserves the original error, which means `errors.Is` and `errors.As` continue to work, regardless of how many times an error has been wrapped. We can also call `errors.Unwrap` to return the previous error in the chain.

wrap还保留了原始错误，这意味着 `errors.Is` 和 `errors.As` 将继续工作，而不管错误被wrap了多少次。我们还可以调用 `errors.Unwrap` 来返回链中的前一个错误。

#### When To Wrap[]（什么时候来Wrap）

Generally, it’s a good idea to wrap an error with at least the function’s name, every time you “bubble it up” - i.e. every time you receive the error from a function and want to continue returning it back up the function chain.

**通常，当层层调用时，最好至少用函数的名称包装错误 - 即每次您从函数收到错误并希望继续将其返回到函数链时**。

There are some exceptions to the rule, however, where wrapping an error may not be appropriate.

但是，有一些例外情况，其中wrap错误可能不合适。

Since wrapping the error always preserves the original error messages, sometimes exposing those underlying issues might be a security, privacy, or even UX concern. In such situations, it could be worth handling the error and returning a new one, rather than wrapping it. This could be the case if you’re writing an open-source library or a REST API where you don’t want the underlying error message to be returned to the 3rd-party user.

由于wrap错误总是保留原始错误消息，有时可能存在暴露安全、隐私甚至用户体验方面的风险。在这种情况下，处理错误并返回一个新的错误可能是值得的，此时应该不是wrap它。如果您正在编写开源库或REST API，而且不希望底层错误消息返回给第三方用户，这可能是情况。

## Conclusion(结论)

That’s a wrap! In summary, here’s the gist of what was covered here:


- Errors in Go are just lightweight pieces of data that implement the `Error` `interface`
- Predefined errors will improve signaling, allowing us to check which error occurred
- Wrap errors to add enough context to trace through function calls (similar to a stack trace)

这就是wrap！总而言之，这里涵盖的内容如下：
- Go 中的错误只是实现了 `Error` 接口的轻量级数据
- 预定义的错误将改进error 传递，使我们能够检查发生了哪个错误
- wrap错误以添加足够的上下文来跟踪函数调用（类似于堆栈跟踪）


## References[](#references)

- [Error handling and Go](https://go.dev/blog/error-handling-and-go)
- [Go 1.13 Errors](https://go.dev/blog/go1.13-errors)
- [Go Error Doc](https://pkg.go.dev/errors@go1.17.5)
- [Go By Example: Errors](https://gobyexample.com/errors)
- [Go By Example: Panic](https://gobyexample.com/errors)

