> 本文是 [Effective Error Handling in Golang](https://earthly.dev/blog/golang-errors/)的中文翻译版本，内容有删减。

其他优秀的Golang error handle 文章：
- [一套优雅的 Go 错误问题解决方案](https://mp.weixin.qq.com/s/RFF2gSikqXiWXIaOxQZsxQ)


Go 中的 Errors 处理与其他主流编程语言（如 Java、JavaScript 或 Python）略有不同。Go 的内置 Errors 不包含堆栈跟踪，也不支持传统的 `try`/`catch` 方法来处理。相反，Go 中的 Errors 只是函数返回的值，它们可以像处理任何其他数据类型一样被处理, 它为 Go 带来了令人惊讶的轻量级和简单的设计。


在本文中，我将从 Go 中处理错误的基础知识开始讲解，同时还有在代码中可以遵循的一些简单策略，以确保您的程序是健壮的和易于调试的。

### Error类型 (The Error Type)


Go 中的错误类型是通过以下接口实现的：

```Golang
type error interface {
    Error() string
}

```

所以，Go 中的错误是实现了 `Error()`的任何方法，该方法返回一个字符串类型的错误消息。就是这么简单！

### 构建Errors (Constructing Errors)

Errors 可以使用 Go 的内置 `errors` 或 `fmt` 包进行构造。例如，以下函数使用 errors 包返回一个带有静态错误消息的 error：

```Golang
package main

import "errors"

func DoSomething() error {
    return errors.New("something didn't work")
}

```

类似地，`fmt` 包可以用于将数据动态添加到 error 中，例如 `int`、`string` 或另一个 `error`。例如：

```Golang
package main

import "fmt"

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("can't divide '%d' by zero", a)
    }
    return a / b, nil
}

```

请注意，当使用 `%w` 格式去 wrap 另一个错误时，`fmt.Errorf` 将非常有用（本文的后面进一步详细介绍）。

上面的例子中有一些其他需要注意的事情。

- Errors 可以返回为 `nil`，事实上，它是 Go 中 error 的默认值或“零”值。这很重要，因为检查 `if err != nil` 是惯用的方法来确定是否遇到错误（替换您可能熟悉的其他编程语言中的 `try`/`catch` 语句）。
- Errors 通常作为函数的最后一个参数返回。因此，在上面的示例中，我们按顺序返回一个 `int` 和一个 `error`。
- 当返回一个 Error 时，函数返回的其他参数通常返回为它们的默认`nil`值。函数的调用者可能希望如果返回了非 `nil` 错误，则返回的其他参数不相关。
- 最后，Errors **通常以小写字母开头，不以标点符号结尾。但在某些情况下也存在例外: 例如包含专有名词、以大写字母开头的函数名等**。


### 按照预期定义错误 (Defining Expected Errors )

另一个 Go 中的重要点是按照预期定义错误，以便可以在代码的其他部分中显式检查它们。当您在遇到某种错误需要执行不同的代码分支时，这很有用。

#### 定义哨兵错误 (Defining Sentinel Errors)

基于前面的 `Divide` 函数，我们可以通过预定义一个“哨兵”错误来改进 Errors 处理, 在其他函数中时可以使用 `errors.Is` 显式检查此错误：

```Golang
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

#### 自定义 Error 类型（Defining Custom Error Types）

大多数的Errors 处理可以采用上述的策略，然而有时您可能需要更多的功能。比如您希望 Errors 携带其他数据字段，或者用动态值填充 Errors 消息。

你可以通过实现自定义错误类型来实现这一点。

下面是前面例子的一个小改动。我们用 `DivisionError`实现了 `Error` `interface`。我们可以使用 `errors.As` 来检查并更具体的 `DivisionError`。


```Golang
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

请注意：您还可以在需要时自定义 `errors.Is` 和 `errors.As` 的行为。可以参考 [this Go.dev blog](https://go.dev/blog/go1.13-errors) 获取示例。

另请注意：`errors.Is` 是在 Go 1.13 版本中添加的，它比使用`err == ...` 更适合。下面会介绍更多。

### 包装 Errors (Wrapping Errors)

目前为止，这些例子中的 Errors 都是在单个函数调用中创建、返回和处理的。换句话说，处理 Errors 的函数调用栈只有一层。

然而在实际的程序中，可能会涉及更多的函数 - 从初始产生 Errors 的函数，到最终处理 Errors 的函数，以及介于两者之间的任意数量的附加函数。

在 Go 1.13 中，引入了几个新的 Errors API，包括 `errors.Wrap` 和 `errors.Unwrap`，它们在将额外的上下文应用于 Errors 时非常有用，它可以在 Errors 被wrap多次时，达到检查特定的 Errors 类型。


>  **历史趣闻** 在 2019 年发布的 Go 1.13 之前，标准库并没有提供许多用于处理 Errors 的 API - 它基本上只是 `errors.New` 和 `fmt.Errorf`。因此，您可能会遇到在野外使用旧版 Go 程序的情况，这些程序没有实现一些较新的 Errors API。许多旧版程序还使用了第三方 Errors 库，例如 [`pkg/errors`](

### 旧版(Before Go 1.13)

通过查看一些旧版 API 的例子，可以很容易地看出 Go 1.13+ 中新的 Errors API 多有用。

让我们考虑一个简单的程序，它管理着用户的数据库。在这个程序中，我们将有一些函数参与到数据库 Errors 的生命周期中。

为了简单起见，让我们将真实的数据库替换为完全“假的”数据库，我们从 `"example.com/fake/users/db"` 导入它

我们还假设这个数据库已经包含了一些用于查找和更新用户记录的函数。并且用户记录被定义为一个结构体：

```Golang
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


```Golang
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

现在，如果我们的数据库操作失败了，会发生什么呢？
`main` 函数中的错误检查应该捕获到这个错误，并打印出类似这样的内容：



```Golang
failed finding or updating user: malformed request

```

但是是哪个数据库操作产生了错误呢？不幸的是，我们在错误日志中没有足够的信息来知道它是来自 `FindUser` 还是 `SetUserAge`。

Go 1.13 引入了一种简单的方法来添加这些信息。

### Errors Are Better Wrapped (更好地包装Errors)

下面的代码片段使用 `fmt.Errorf` 和 `%w`进行了重构，通过其他函数层层调用来“wrap”错误 。通过添加调用的上下文信息，可以推断出在上一示例中哪些数据库操作失败。

```go
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

现在，我们的Error消息包含了足够的信息，我们可以看到问题是在 `db.SetUserAge` 函数中产生的。唷！这无疑为我们节省了一些调试时间！


如果使用得当，错误wrap可以以类似于传统堆栈跟踪的方式提供有关错误谱系的附加上下文。

wrap还保留了原始错误，这意味着 `errors.Is` 和 `errors.As` 将继续工作，而不管错误被wrap了多少次。我们还可以调用 `errors.Unwrap` 来返回链中的前一个错误。

#### 什么时候用Wrap（When To Wrap）

**通常，当层层调用时，最好至少用函数的名称包装错误 - 即每次您从函数收到错误并希望继续将其返回到函数链时**。


但是，有一些例外情况，其中wrap错误可能不合适。

由于wrap错误总是保留原始错误消息，有时可能存在暴露安全、隐私甚至用户体验方面的风险。在这种情况下，处理错误并返回一个新的错误可能是值得的，此时应该不是wrap它。如果您正在编写开源库或REST API，而且不希望底层错误消息返回给第三方用户，这可能是情况。

### 结论 (Conclusion)

这就是wrap！总而言之，这里涵盖的内容如下：
- Go 中的错误只是实现了 `Error` 接口的轻量级数据
- 预定义的错误将改进error 传递，使我们能够检查发生了哪个错误
- wrap错误以添加足够的上下文来跟踪函数调用（类似于堆栈跟踪）


### References[](#references)

- [Error handling and Go](https://go.dev/blog/error-handling-and-go)
- [Go 1.13 Errors](https://go.dev/blog/go1.13-errors)
- [Go Error Doc](https://pkg.go.dev/errors@go1.17.5)
- [Go By Example: Errors](https://gobyexample.com/errors)
- [Go By Example: Panic](https://gobyexample.com/errors)

