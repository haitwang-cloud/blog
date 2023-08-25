> 本文是[cobra的官方readme](https://github.com/spf13/cobra/blob/main/site/content/user_guide.md)的中文翻译版本

# 用户指南

当你想要用cobra来构建自己的应用时，你可以按照下面的结构来组织你的cobra应用：

```
  ▾ appName/
    ▾ cmd/
        add.go
        your.go
        commands.go
        here.go
      main.go
```

在一个cobra应用中，通常main.go文件非常简单，它只有一个目的：初始化cobra。

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

## 使用Cobra Generator

Cobra-CLI 是一个独立的程序，它将创建您的应用程序并添加您想要的任何命令。
这是将 Cobra 集成到您的应用程序的最简单方法。


关于使用Cobra generator的细节，可以参考[The Cobra-CLI Generator README](https://github.com/spf13/cobra-cli/blob/main/README.md)

## 使用Cobra库


要实现 Cobra，您需要创建一个空的 `main.go` 文件和一个 `rootCmd` 文件。
您可以根据需要选择性地提供其他命令。

### 创建rootCmd


Cobra 不需要任何特殊的构造函数。只需创建命令即可。

您可以把下面的内容放在 app/cmd/root.go 中。

```go
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io/documentation/`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```
您还将在 `init()` 函数中定义`flag`和对应的处理函数。

例如，在 `cmd/root.go` 文件中可以这样写：


```go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

var (
	// Used for flags.
	cfgFile     string
	userLicense string

	rootCmd = &cobra.Command{
		Use:   "cobra-cli",
		Short: "A generator for Cobra based Applications",
		Long: `Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	}
)

// Execute executes the root command.
func Execute() error {
	return rootCmd.Execute()
}

func init() {
	cobra.OnInitialize(initConfig)

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
	rootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "author name for copyright attribution")
	rootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "name of license for the project")
	rootCmd.PersistentFlags().Bool("viper", true, "use Viper for configuration")
	viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
	viper.BindPFlag("useViper", rootCmd.PersistentFlags().Lookup("viper"))
	viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
	viper.SetDefault("license", "apache")

	rootCmd.AddCommand(addCmd)
	rootCmd.AddCommand(initCmd)
}

func initConfig() {
	if cfgFile != "" {
		// Use config file from the flag.
		viper.SetConfigFile(cfgFile)
	} else {
		// Find home directory.
		home, err := os.UserHomeDir()
		cobra.CheckErr(err)

		// Search config in home directory with name ".cobra" (without extension).
		viper.AddConfigPath(home)
		viper.SetConfigType("yaml")
		viper.SetConfigName(".cobra")
	}

	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err == nil {
		fmt.Println("Using config file:", viper.ConfigFileUsed())
	}
}
```

### 创建你的main.go文件

你需要在你的main函数中执行root命令。为了清晰起见，应该在root上执行Execute（其实它可以在任何命令上执行）。

在一个Cobra应用中，通常main.go文件非常简单，它只有一个目的：初始化Cobra。

```go
package main

import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

### 创建其他命令

> 您可以在cmd/目录下的文件中单独定义和创建其他命令。



如果你想创建一个version命令，你可以创建cmd/version.go文件，并写上以下内容：


```go
package cmd

import (
  "fmt"

  "github.com/spf13/cobra"
)

func init() {
  rootCmd.AddCommand(versionCmd)
}

var versionCmd = &cobra.Command{
  Use:   "version",
  Short: "Print the version number of Hugo",
  Long:  `All software has versions. This is Hugo's`,
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
  },
}
```

### 添加子命令
一个命令可以有子命令，子命令也可以有其他子命令。这可以通过使用 `AddCommand` 来实现。在某些情况下，尤其是在较大的应用程序中，每个子命令可能定义在自己的 Go 包中。

建议的方法是父命令使用 `AddCommand` 来添加其最直接的子命令。例如，考虑以下目录结构：

```text
├── cmd
│   ├── root.go
│   └── sub1
│       ├── sub1.go
│       └── sub2
│           ├── leafA.go
│           ├── leafB.go
│           └── sub2.go
└── main.go
```

在这种情况下：

* `root.go`的`init`函数将`sub1.go`中定义的命令添加到root命令。
* `sub1.go`的`init`函数将`sub2.go`中定义的命令添加到sub1命令。
* `sub2.go`的`init`函数将`leafA.go`和`leafB.go`中定义的命令添加到sub2命令。

这个方法确保了子命令在编译时被包含进来，同时避免了循环引用的发生。

### 返回和处理错误

If you wish to return an error to the caller of a command, `RunE` can be used.
如果您想将错误返回给命令的调用方，可以使用 `RunE`。

```go
package cmd

import (
  "fmt"

  "github.com/spf13/cobra"
)

func init() {
  rootCmd.AddCommand(tryCmd)
}

var tryCmd = &cobra.Command{
  Use:   "try",
  Short: "Try and possibly fail at something",
  RunE: func(cmd *cobra.Command, args []string) error {
    if err := someFunc(); err != nil {
	return err
    }
    return nil
  },
}
```
上面的错误可以在执行函数调用时被捕获。


## 使用flag Flags


Flags 提供了修改器来控制命令的操作。

### 在命令中使用flag
由于flag是在不同的位置定义和使用的，因此我们需要在外部定义一个变量，来为它分配flag进行操作。


```go
var Verbose bool
var Source string
```

有两种不同的方法来使用一个flag。

### 全局flag（持久化flag）

一个flag可以是“永久的”，这意味着该flag将对它所分配的命令以及该命令下的所有命令都有效。对于全局flag，请在root命令上将flag分配为永久flag。


```go
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")
```

### 局部flag

一个flag也可以被分配为局部flag，这将只适用于该特定命令。

```go
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

### 从父命令中获取局部flag

默认情况下，Cobra只解析命令上的本地flag，父命令上的任何本地flag都会被忽略。通过启用 `Command.TraverseChildren`，Cobra将在执行目标命令之前解析每个命令上的本地flag。

```go
command := cobra.Command{
  Use: "print [OPTIONS] [COMMANDS]",
  TraverseChildren: true,
}
```

### 使用config绑定flag

You can also bind your flags with [viper](https://github.com/spf13/viper): 您也可以使用[viper](https://github.com/spf13/viper)来绑定您的flag：

```go
var author string

func init() {
  rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```

在本例中，永久flag `author` 使用 `viper` 绑定。
> 注意：当用户提供 `--author` 标志时，变量 `author` 将不会被设置为配置文件中的值

详情可以参考[viper文档](https://github.com/spf13/viper#working-with-flags).

### 必需的flag

默认情况下，flag是可选的。如果您希望您的命令在flag未设置时报告错误，可以将其标记为必需的：

```go
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```
或者，对于持久flag：

```go
rootCmd.PersistentFlags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkPersistentFlagRequired("region")
```

### Flag 组
如果你有不同的flag必须一起使用（例如，如果用户提供了 `--username` flag，他们必须同时提供 `--password` flag），那么Cobra可以通过下面的方式来做：

```go
rootCmd.Flags().StringVarP(&u, "username", "u", "", "Username (required if password is set)")
rootCmd.Flags().StringVarP(&pw, "password", "p", "", "Password (required if username is set)")
rootCmd.MarkFlagsRequiredTogether("username", "password")
```

如果不同的flag代表互斥的选项，你可以这样做。例如指定一个输出格式为 `--json` 或 `--yaml`，但从不同时使用：
```go
rootCmd.Flags().BoolVar(&ofJson, "json", false, "Output in JSON")
rootCmd.Flags().BoolVar(&ofYaml, "yaml", false, "Output in YAML")
rootCmd.MarkFlagsMutuallyExclusive("json", "yaml")
```

如果你想要至少一个flag必须存在，你可以使用 `MarkFlagsOneRequired`。这可以与 `MarkFlagsMutuallyExclusive` 结合使用，以强制执行给定组中的一个flag：

```go
rootCmd.Flags().BoolVar(&ofJson, "json", false, "Output in JSON")
rootCmd.Flags().BoolVar(&ofYaml, "yaml", false, "Output in YAML")
rootCmd.MarkFlagsOneRequired("json", "yaml")
rootCmd.MarkFlagsMutuallyExclusive("json", "yaml")
```

>在这些情况下：
>  - 本地和持久flag都可以使用
>    - **注意：**该组仅在定义了每个flag的命令上强制执行
>  - 一个flag可以出现在多个组中
>  - 一个组可以包含任意数量的flag

## 潜在的参数和自定义的参数
潜在参数的验证可以使用`Command`的`Args`字段来指定。
以下验证器是内置的：
- 参数的数量：
  - `NoArgs` - 如果有任何位置参数，则报告错误。
  - `ArbitraryArgs` - 接受任意数量的参数。
  - `MinimumNArgs(int)` - 如果提供的位置参数少于N个，则报告错误。
  - `MaximumNArgs(int)` - 如果提供的位置参数多于N个，则报告错误。
  - `ExactArgs(int)` - 如果位置参数不是N个，则报告错误。
  - `RangeArgs(min, max)` - 如果参数的数量不在`min`和`max`之间，则报告错误。


如果`Args`未定义或为`nil`，则默认为`ArbitraryArgs`。

此外，`MatchAll(pargs ...PositionalArgs)` 允许将现有检查与任意其他检查组合。
例如，如果您想在没有确切的 N 个位置参数时报告错误，或在任何位置参数不在 `Command` 的 `ValidArgs` 字段中时报告错误，您可以调用 `MatchAll` 方法，并将 `ExactArgs` 和 `OnlyValidArgs` 方法作为参数传递


```go
var cmd = &cobra.Command{
  Short: "hello",
  Args: cobra.MatchAll(cobra.ExactArgs(2), cobra.OnlyValidArgs),
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

可以设置任何满足以下签名的自定义验证器：
例如

```go
var cmd = &cobra.Command{
  Short: "hello",
  Args: func(cmd *cobra.Command, args []string) error {
    // Optionally run one of the validators provided by cobra
    if err := cobra.MinimumNArgs(1)(cmd, args); err != nil {
        return err
    }
    // Run the custom validation logic
    if myapp.IsValidColor(args[0]) {
      return nil
    }
    return fmt.Errorf("invalid color specified: %s", args[0])
  },
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hello, World!")
  },
}
```

## Example 例子

在下面的例子中，我们定义了三个命令。两个在顶层，一个（cmdTimes）是顶级命令之一的子命令。在这种情况下，根命令不可执行，这意味着需要子命令。这是通过不为 `rootCmd` 提供 `Run` 方法来实现的。

我们只为单个命令定义了一个标志。

有关标志的更多文档可在 https://github.com/spf13/pflag 上找到

```go
package main

import (
  "fmt"
  "strings"

  "github.com/spf13/cobra"
)

func main() {
  var echoTimes int

  var cmdPrint = &cobra.Command{
    Use:   "print [string to print]",
    Short: "Print anything to the screen",
    Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Print: " + strings.Join(args, " "))
    },
  }

  var cmdEcho = &cobra.Command{
    Use:   "echo [string to echo]",
    Short: "Echo anything to the screen",
    Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Println("Echo: " + strings.Join(args, " "))
    },
  }

  var cmdTimes = &cobra.Command{
    Use:   "times [string to echo]",
    Short: "Echo anything to the screen more times",
    Long: `echo things multiple times back to the user by providing
a count and a string.`,
    Args: cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
      for i := 0; i < echoTimes; i++ {
        fmt.Println("Echo: " + strings.Join(args, " "))
      }
    },
  }

  cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

  var rootCmd = &cobra.Command{Use: "app"}
  rootCmd.AddCommand(cmdPrint, cmdEcho)
  cmdEcho.AddCommand(cmdTimes)
  rootCmd.Execute()
}
```

如果你想要一个更完整的例子，可以看看[Hugo](https://gohugo.io/)。

## Help 命令


Cobra 会自动在你的应用程序中添加一个 help 命令，当用户运行 `app help`时会调用它。此外，help 还支持所有其他命令作为输入。例如，假设你有一个名为 create 的命令，没有任何其他配置；当调用 `app help create`时，Cobra 将会正常工作。每个命令都会自动添加 `--help` 标志。


### 例子


下面的输出是由Cobra自动生成的。除了命令和flag定义之外，不需要任何其他内容。
```shell
    $ cobra-cli help

    Cobra is a CLI library for Go that empowers applications.
    This application is a tool to generate the needed files
    to quickly create a Cobra application.

    Usage:
      cobra-cli [command]

    Available Commands:
      add         Add a command to a Cobra Application
      completion  Generate the autocompletion script for the specified shell
      help        Help about any command
      init        Initialize a Cobra Application

    Flags:
      -a, --author string    author name for copyright attribution (default "YOUR NAME")
          --config string    config file (default is $HOME/.cobra.yaml)
      -h, --help             help for cobra-cli
      -l, --license string   name of license for the project
          --viper            use Viper for configuration

    Use "cobra-cli [command] --help" for more information about a command.

```
Help 只是一个像其他命令一样的命令。它没有特殊的逻辑或行为。事实上，如果你想的话，你可以提供你自己的help。

### Help中的命令分组

Cobra 支持在帮助输出中对可用命令进行分组。要对命令进行分组，每个组都必须显式定义，使用 `AddGroup()` 方法在父命令上。然后，可以使用该子命令的 `GroupID` 元素将子命令添加到组中。

这些组将在帮助输出中以与使用不同 `AddGroup()` 调用定义的顺序相同的顺序出现。

如果您使用生成的 `help` 或 `completion` 命令，则可以使用 `SetHelpCommandGroupId()` 和 `SetCompletionCommandGroupId()` 方法分别在根命令上设置它们的组 ID。

### 定义你自己的help

你可以使用以下函数调用你自己的 Help 命令或你自己的模板，供默认命令使用：

```go
cmd.SetHelpCommand(cmd *Command)
cmd.SetHelpFunc(f func(*Command, []string))
cmd.SetHelpTemplate(s string)
```
后两个也将应用于任何子命令。

## 用法信息

当用户提供无效的flag或无效的命令时，Cobra会通过显示用户的“用法”来做出响应。

### 例子

```shell
    $ cobra-cli --invalid
    Error: unknown flag: --invalid
    Usage:
      cobra-cli [command]

    Available Commands:
      add         Add a command to a Cobra Application
      completion  Generate the autocompletion script for the specified shell
      help        Help about any command
      init        Initialize a Cobra Application

    Flags:
      -a, --author string    author name for copyright attribution (default "YOUR NAME")
          --config string    config file (default is $HOME/.cobra.yaml)
      -h, --help             help for cobra-cli
      -l, --license string   name of license for the project
          --viper            use Viper for configuration

    Use "cobra [command] --help" for more information about a command.
```
### 自定义用法

你可以为Cobra提供自己的用法函数或模板。与help一样，函数和模板可以通过公共方法进行覆盖：

```go
cmd.SetUsageFunc(f func(*Command) error)
cmd.SetUsageTemplate(s string)
```

## 版本Flag

Cobra 会在根命令上设置 Version 字段时添加一个全局标志 `--version`。

使用 `--version` 标志运行应用程序将使用版本模板将版本打印到标准输出。可以使用 `cmd.SetVersionTemplate(s string)` 函数自定义模板。


## PreRun 和 PostRun 钩子 

可以运行函数在你的命令的主 `Run` 函数之前或之后。`PersistentPreRun` 和 `PreRun` 函数将在 `Run` 之前执行。`PersistentPostRun` 和 `PostRun` 函数将在 `Run` 之后执行。如果子命令没有声明自己的，则 `Persistent*Run` 函数将被继承。这些函数按照以下顺序运行

- `PersistentPreRun`
- `PreRun`
- `Run`
- `PostRun`
- `PersistentPostRun`

下面是两个命令的例子，它们使用了所有这些特性。当执行子命令时，它将运行根命令的 `PersistentPreRun`，但不会运行根命令的 `PersistentPostRun`：

```go
package main

import (
  "fmt"

  "github.com/spf13/cobra"
)

func main() {

  var rootCmd = &cobra.Command{
    Use:   "root [sub]",
    Short: "My root command",
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPreRun with args: %v\n", args)
    },
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside rootCmd PersistentPostRun with args: %v\n", args)
    },
  }

  var subCmd = &cobra.Command{
    Use:   "sub [no options!]",
    Short: "My subcommand",
    PreRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PreRun with args: %v\n", args)
    },
    Run: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd Run with args: %v\n", args)
    },
    PostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PostRun with args: %v\n", args)
    },
    PersistentPostRun: func(cmd *cobra.Command, args []string) {
      fmt.Printf("Inside subCmd PersistentPostRun with args: %v\n", args)
    },
  }

  rootCmd.AddCommand(subCmd)

  rootCmd.SetArgs([]string{""})
  rootCmd.Execute()
  fmt.Println()
  rootCmd.SetArgs([]string{"sub", "arg1", "arg2"})
  rootCmd.Execute()
}
```

Output:
```
Inside rootCmd PersistentPreRun with args: []
Inside rootCmd PreRun with args: []
Inside rootCmd Run with args: []
Inside rootCmd PostRun with args: []
Inside rootCmd PersistentPostRun with args: []

Inside rootCmd PersistentPreRun with args: [arg1 arg2]
Inside subCmd PreRun with args: [arg1 arg2]
Inside subCmd Run with args: [arg1 arg2]
Inside subCmd PostRun with args: [arg1 arg2]
Inside subCmd PersistentPostRun with args: [arg1 arg2]
```

##  当发生“未知命令”时的提示

Cobra在发生“未知命令”错误时会打印自动建议。这使得Cobra在发生拼写错误时的行为类似于`git`命令。例如：

```shell
$ hugo srever
Error: unknown command "srever" for "hugo"

Did you mean this?
        server

Run 'hugo --help' for usage.
```

提示是基于现有的子命令自动生成的，并使用[Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance)的实现。每个注册的命令，如果匹配的最小距离为2（忽略大小写），将被显示为一个提示。

如果您需要在命令中禁用建议或调整字符串距离，请使用：

```go
command.DisableSuggestions = true
```

或者

```go
command.SuggestionsMinimumDistance = 1
```

You can also explicitly set names for which a given command will be suggested using the `SuggestFor` attribute. This allows suggestions for strings that are not close in terms of string distance, but make sense in your set of commands but for which
you don't want aliases. Example:

你可以使用 `SuggestFor` 属性显式地设置命令将被建议的名称。这允许为在字符串距离方面不接近但在命令集中有意义的字符串提供建议，例如：


```shell
$ kubectl remove
Error: unknown command "remove" for "kubectl"

Did you mean this?
        delete

Run 'kubectl help' for usage.
```
