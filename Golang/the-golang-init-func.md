> [The Go init Function](https://tutorialedge.net/golang/the-go-init-function/) 的中文翻译版本。


There are times, when creating applications in Go, that you need to be able to set up some form of state on the initial startup of your program. This could involve creating connections to databases, or loading in configuration from locally stored configuration files.

当使用 Go 创建应用程序时，有的时候您需要能够在程序启动时初始化某些资源。例如涉及创建数据库的连接，或从本地存储的配置文件加载配置。

In this tutorial, we’ll be looking at how you can use this `init()` function to achieve this goal and we’ll also be taking a look at why this might not necessarily be the best approach to instantiating your components.

在本教程中，我们将研究如何使用这个 `init()` 函数来实现初始化，我们还将看看为什么这不一定是实例化组件的最佳方法。

Alternative Approaches 
（替代方案）
----------------------

Now, the typical use-case for using the `init` function might go along the lines of “I want to instantiate a connection to the database”. However, this might actually be a side-effect of poor design within you Go applications.

现在，使用 `init` 函数的典型用例可能类似于“我想实例化与数据库的连接”。但是这实际上可能是 Go 应用程序中设计不佳的副作用。

A better approach to instantiating things like database connections might instead be to use a `New` function that returns a pointer to a struct containing your database connection object.

实例化数据库连接之类的更好方法可能是使用“New”函数，该函数返回指向包含数据库连接对象的结构的指针。

```
func New() (*Database, error) {
    conn, err := sql.Open("postgres", "connectionURI")
    if err != nil {
        return &Database{}, err
    }
    return &Database{
        Connection: conn,
    }, nil
}
```

With this approach, you would then be able to pass in your database to any other components in your system that may need to call the database.

使用这种方法，您可以将数据库传递给系统中可能需要调用数据库的任何其他组件。

It also gives you more control over how you handle failures during startup as opposed to simply terminating your application.

它还使您可以在启动期间如何更好地处理故障，而不是简单地终止您的应用程序。

Overall, I would try and generally avoid using the `init` function where possible and use the approach outlined above to build your Go applications.

总的来说，我会尽量避免使用 `init` 函数，并使用上面概述的方法来实例化数据库连接，构建Go 应用程序。

The init Function
-----------------

In Go, the `init()` function can be incredibly powerful and compared to some other languages, is a lot easier to use within your Go programs. These `init()` functions can be used within a `package` block and regardless of how many times that package is imported, the `init()` function will only be called once.

在 Go 中，`init()` 函数非常强大。同时与其他一些语言相比，在 Go 程序中更容易使用。这些 `init()` 函数可以在 `package` 块中使用，并且无论该包被导入多少次，`init()` 函数只会被调用一次。

Now, the fact that it is only called once is something you should pay close attention to. This effectively allows us to set up database connections, or register with various service registries, or perform any number of other tasks that you typically only want to do once.

**现在你需要知道的是，`init()` 函数只会被调用一次**。当我们想要建立数据库连接，或向各种服务注册中心注册，或执行您通常只想执行一次的任何数量的其他任务，它会是十分有效的。

```
package main

func init() {
  fmt.Println("This will get called on main initialization")
}

func main() {
  fmt.Println("My Wonderful Go Program")
}
```

Notice in this above example, we’ve not explicitly called the `init()` function anywhere within our program. Go handles the execution for us implicitly and thus the above program should provide output that looks like this:

请注意在上面的示例中，我们没有在程序中的任何地方显式调用 `init()` 函数。 Go 隐式地为我们处理执行，因此上面的程序输出如下：

```
$ go run test.go
This will get called on main initialization
My Wonderful Go Program
```

Awesome, so with this working, we can start to do cool things such as variable initialization.

通过这个工作，我们可以开始做一些很酷的事情，比如变量初始化。

```
package main

import "fmt"

var name string

func init() {
    fmt.Println("This will get called on main initialization")
    name = "Elliot"
}

func main() {
    fmt.Println("My Wonderful Go Program")
    fmt.Printf("Name: %s\n", name)
}
```

In this example, we can start to see why using the `init()` function would be preferential when compared to having to explicitly call your own setup functions.

在这个例子中，我们可以开始了解为什么使用 `init()` 函数比显式调用您自己的setup函数相比会更受欢迎。

When we run the above program, you should see that our `name` variable has been properly set and whilst it’s not the most useful variable on the planet, we can certainly still use it throughout our Go program.

当我们运行上面的程序时，你应该会看到我们的 `name` 变量已经正确赋值。虽然它不是这个星球上最有用的变量😊，但我们当然仍然可以在整个 Go 程序中使用它。

```
$ go run test.go
This will get called on main initialization
My Wonderful Go Program
Name: Elliot
```

Multiple Packages（多个包的情况）
-----------------

Let’s have a look at a more complex scenario that is closer to what you’d expect in a production Go system. Imagine we had 4 distinct Go packages within our application, `main`, `broker`, and `database`.

接下来是更接近在生产 Go 系统中更复杂的场景，比如我们的应用程序中有 4 个不同的 Go 包，“main”、“broker”和“database”。

In each of these we could specify an `init()` function which would perform the task of setting up the connection pool to the various 3rd party services such as Kafka or MySQL.

在这些包中，我们可以指定一个 `init()` 函数，它将执行以下操作：各种第三方服务（如 Kafka 或 MySQL）设置连接池的任务。

Whenever we then call a function in our `database` package, it would then use the connection pool that we set up in our `init()` function.

当你调用了了 `database` 包中任何函数的时候，它都会使用我们在 `init()` 函数中设置的连接池。

> **Note -** It’s incredibly important to note that you cannot rely upon the order of execution of your `init()` functions. It’s instead better to focus on writing your systems in such a way that the order does not matter.

> **Note -** 请注意，你不能依赖于你的 `init()` 函数的执行顺序。更好的选择是考虑写一个系统的方式，使得执行顺序不重要。

Order of Initialization
（初始化顺序）
-----------------------

For more complex systems, you may have more than one file making up any given package. Each of these files may indeed have their own `init()` functions present within them. So how does Go order the initialization of these packages?

对于更加复杂的系统，你可能有多个文件组成一个包。这些文件可能有自己的 `init()` 函数。所以 Go 如何确定这些包的初始化顺序呢？

When it comes to the order of the initialization, a few things are taken into consideration. Things in Go are typically initialized in the order in declaration order but explicitly after any variables they may depend on. This means that, should you have 2 files `a.go` and `b.go` in the same package, if the initialization of anything in `a.go` depends on things in `b.go` they will be initialized first.

当涉及到初始化的顺序时，需要考虑一些事情。Go 通常是按照声明顺序初始化，但是如果它们依赖于其他文件，那么它们必定将会先初始化。这意味着如果你在同一个包中有 2 个文件 `a.go` 和 `b.go` ，如果 `a.go` 中的任何东西依赖于 `b.go` 中的东西，它们（`a.go`）将会先初始化。


> **Note -** A more in-depth overview of the order of initialization in Go can be found in the official docs: [Package Initialization](https://golang.org/ref/spec#Package_initialization)

> **Note -** 你可以在官方文档中找到有关 Go 中初始化顺序的更深入概述：: [Package Initialization](https://golang.org/ref/spec#Package_initialization)


The key point to note from this is this order of declaration can lead to scenarios such as this:

需要注意的是，这个顺序可能会导致这样的场景：

```
// source: https://stackoverflow.com/questions/24790175/when-is-the-init-function-run
var WhatIsThe = AnswerToLife()

func AnswerToLife() int {
    return 42
}

func init() {
    WhatIsThe = 0
}

func main() {
    if WhatIsThe == 0 {
        fmt.Println("It's all a lie.")
    }
}
```

In this scenario, you’ll see that `AnswerToLife()` will run before our `init()` function as our `WhatIsThe` variable is declared before our `init()` function is called.

```
$ go run main.go
                                        
It's all a lie.
```

在这个场景中，你会看到 `AnswerToLife()` 在我们的 `init()` 函数之前被调用。因为我们的 `WhatIsThe` 变量被声明在我们的 `init()` 函数之前，所以它的值是 `0`。

Multiple Init Functions in the Same File
（在同一个文件中有多个初始化函数）
----------------------------------------

What happens if we have multiple `init()` functions within the same Go file? At first I didn’t think this was possible, but Go does indeed support having 2 separate `init()` functions within the one file.

如果我们在同一个 Go 文件中有多个 `init()` 函数会发生什么？首先我不会认为这是可能的，但是 Go 却实际上可以支持有 2 个不同的 `init()` 函数在同一个文件中。


These `init()` functions are again called in their respective order of declaration within the file:

这些 `init()` 函数仍然会按照声明顺序在文件中被调用：

```
package main

import "fmt"

// this variable is initialized first due to
// order of declaration
var initCounter int

func init() {
    fmt.Println("Called First in Order of Declaration")
    initCounter++
}

func init() {
    fmt.Println("Called second in order of declaration")
    initCounter++
}

func main() {
    fmt.Println("Does nothing of any significance")
    fmt.Printf("Init Counter: %d\n", initCounter)
}
```

Now, when you run the above program, you should see the output look like so:

现在当你运行上面的程序时，你会看到输出如下：

```
$ go run test.go
Called First in Order of Declaration
Called second in order of declaration
Does nothing of any significance
Init Counter: 2
```

Pretty cool, huh? But what is this for? Why do we allow this? Well, for more complex systems, it allows us to break up complex initialization procedures into multiple, easier-to-digest `init()` functions. It essentially allows us to avoid having one monolithic code block in a single `init()` function which is always a good thing. The one caveat of this style is that you will have to take care when ensuring declaration order.

很酷吧？但这是为了什么？为什么我们允许这样做？好吧，对于更复杂的系统，Go 允许我们将复杂的初始化过程分解为多个更易于理解的 `init()` 函数。Go本质上允许我们避免在单个 `init()` 函数中有一个单一的代码块，这是一件好事。但是，这种方式的一个缺点是，你需要注意声明顺序。


Conclusion
----------

So this concludes the basic introduction to the world of `init()` functions. Once you’ve mastered the use of the package initialization, you may find it useful to master the art of structuring your Go based projects.

至此，对 `init()` 函数世界的基本介绍到此结束。一旦你掌握了包初始化的使用，可能会对你掌握 Go 项目的结构化有所帮助。