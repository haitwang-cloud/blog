> 本文是 [Prefer table driven tests](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests)的中文翻译版本，内容有删减



我是编写测试代码的狂热粉丝，特别喜欢[unit testing](https://dave.cheney.net/2019/04/03/absolute-unit-test)和[TDD](https://www.youtube.com/watch?v=EZ05e7EMOLM)。时下在Go项目中比较流行的是表格驱动测试，本文将会讨论如何编写针对Go的表格驱动测试


假设我们有一个函数用于分割字符串：

```go
// Split slices s into all substrings separated by sep and
// returns a slice of the substrings between those separators.
func Split(s, sep string) []string {
    var result []string
    i := strings.Index(s, sep)
    for i > -1 {
        result = append(result, s[:i])
        s = s[i+len(sep):]
        i = strings.Index(s, sep)
    }
    return append(result, s)
}

```
## 单元测试
在Go中，单元测试就是普通的Go函数（有一些规则），所以我们可以在同一个目录下编写一个单元测试文件，包名为`strings`。

```go
package split

import (
    "reflect"
    "testing"
)

func TestSplit(t *testing.T) {
    got := Split("a/b/c", "/")
    want := []string{"a", "b", "c"}
    if !reflect.DeepEqual(want, got) {
         t.Fatalf("expected: %v, got: %v", want, got)
    }
}

```

编写测试代码和编写普通的Go函数一样，只有两个规则：

1.  测试函数必须以`Test`开头.
2.  测试函数必须接受一个类型为`*testing.T`的参数，`*testing.T`是由测试包自动注入的类型，用于提供打印，跳过和失败测试的方法。
   

在我们的测试中，我们使用一些输入调用`Split`，然后将其与我们预期的结果进行比较。

### 代码覆盖率


接下来的问题是，这个包的覆盖率是多少？幸运的是，go工具有一个内置的分支覆盖率。我们可以像这样调用它：
```bash
% go test -coverprofile=c.out
PASS
coverage: 100.0% of statements
ok      split   0.010s

```

它将会告诉我们，我们有100%的覆盖率，这并不奇怪，因为这段代码只有一个分支。
如果我们想深入了解覆盖率报告，go工具有几个选项可以打印覆盖率报告。我们可以使用`go tool cover -func`来分解每个函数的覆盖率：

```bash
% go tool cover -func=c.out
split/split.go:8:       Split          100.0%
total:                  (statements)   100.0%

```

上面的结果并没有什么激动人心的，因为我们只有一个函数，但是我相信你会找到更多有趣的包来测试。

### 配置.bashrc文件

上面的命令其实非常有用，所以我配置了一个shell alias，可以在一个命令中运行测试覆盖率和报告：

```bash
cover () {
    local t=$(mktemp -t cover)
    go test $COVERFLAGS -coverprofile=$t $@ \
        && go tool cover -func=$t \
        && unlink $t
}

```

### 实现 100% 测试覆盖率

所以现在我们编写了一个测试用例，覆盖率达到了100%，但是这并不是故事的结束。我们有了很好的分支覆盖率，但是我们可能需要测试一些边界条件。例如，如果我们尝试在逗号上进行分割会发生什么？

```go
func TestSplitWrongSep(t *testing.T) {
    got := Split("a/b/c", ",")
    want := []string{"a/b/c"}
    if !reflect.DeepEqual(want, got) {
        t.Fatalf("expected: %v, got: %v", want, got)
    }
}

```
或者，如果源字符串中没有分隔符会发生什么？

```go
func TestSplitNoSep(t *testing.T) {
    got := Split("abc", "/")
    want := []string{"abc"}
    if !reflect.DeepEqual(want, got) {
        t.Fatalf("expected: %v, got: %v", want, got)
    }
}

```


现在，我们开始构建一组测试用例来测试边界条件。

## 表格驱动测试

然而，我们的测试中有很多重复的内容。对于每个测试用例，只有输入，预期输出和测试用例的名称发生了变化。其他所有内容都是样板。我们想要的是设置所有的输入和预期的输出，并将它们传递给单个测试框架。这是一个很好的时机来介绍表驱动测试。

```go
func TestSplit(t *testing.T) {
    type test struct {
        input string
        sep   string
        want  []string
    }

    tests := []test{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
    }

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("expected: %v, got: %v", tc.want, got)
        }
    }
}

```
我们定义了一个结构来保存我们的测试输入和预期输出。这是我们的`表格`。我们把`tests`结构定义为一个局部变量，因为我们希望在这个包中重用这个名字。

实际上，我们甚至不需要给类型一个名字，我们可以使用匿名结构字面量来减少样板，就像这样：

```go
func TestSplit(t *testing.T) {
    tests := []struct {
        input string
        sep   string
        want  []string
    }{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
    } 

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("expected: %v, got: %v", tc.want, got)
        }
    }
}

```
现在，添加一个新的测试只需在`tests`结构中添加另一行即可。让我们来猜一下，如果我们的输入字符串有一个尾随分隔符会发生什么？

```go
{input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
{input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
{input: "abc", sep: "/", want: []string{"abc"}},
{input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}}, // trailing sep

```

当我们运行`go test`时，我们得到下面的结果

```go
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:24: expected: [a b c], got: [a b c ]
```


让我们把失败的测试用例放在一边，有几个问题需要讨论。

第一个问题是，通过将每个测试从函数重写为表中的一行，我们丢失了失败测试的名称。我们在测试文件中添加了一个注释来调用这个案例，但是我们在`go test`输出中没有访问该注释。


其实有几种方法可以解决这个问题。你会看到在Go代码库中使用了一些混合风格，因为表格测试习惯正在不断发展，人们继续尝试这种形式。

### 枚举测试用例

由于测试用例存储在一个切片中，我们可以在失败的消息中打印出测试用例的索引：

```go
func TestSplit(t *testing.T) {
    tests := []struct {
        input string
        sep   string
        want  []string
    }{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
        {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for i, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("test %d: expected: %v, got: %v", i+1, tc.want, got)
        }
    }
}

```
现在运行`go test`的结果是

```bash
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:24: test 4: expected: [a b c], got: [a b c ]


```

情况变得更好了，现在我们知道第四个测试失败了，尽管我们必须做一些小动作，因为切片索引和范围迭代是从零开始的。这需要在测试用例中保持一致；如果有时是从0开始，而其他一些情况则从1开始，那么这将是令人困惑的。而且，如果测试用例的列表很长，那么很难计算大括号，以确定哪个测试用例是第四个。

### 为测试用例命名
另一个常见的模式是在测试struct中声明一个名称字段。


```go
func TestSplit(t *testing.T) {
    tests := []struct {
        name  string
        input string
        sep   string
        want  []string
    }{
        {name: "simple", input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {name: "wrong sep", input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {name: "no sep", input: "abc", sep: "/", want: []string{"abc"}},
        {name: "trailing sep", input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("%s: expected: %v, got: %v", tc.name, tc.want, got)
        }
    }
}

```

现在，当测试失败时，我们有一个描述性的名称，说明测试在做什么。我们不再需要尝试从输出中弄清楚它，因为现在我们有一个可以搜索的字符串。

```bash
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:25: trailing sep: expected: [a b c], got: [a b c ]


```
我们可以使用map字面量语法来进一步简化它：

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{ 
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}}, 
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("%s: expected: %v, got: %v", name, tc.want, got)
        }
    }
}

```

使用map字面量语法，我们将测试用例定义为struct的切片，而是将测试名称定义为测试fixture的map。使用map的一个好处是，它将潜在地提高我们测试的效率。


Map的迭代顺序是无序的。这意味着每次运行`go test`时，我们的测试都有可能以不同的顺序运行。

这个其实很有用，可以用来发现测试在语句顺序运行时通过，但在其他情况下不通过的情况。如果发现这种情况，那么你可能有一些全局状态，这些状态被一个测试修改，后续的测试依赖于这个修改。

## 介绍子测试

在我们修复失败的测试之前，我们的表驱动测试框架还有一些其他问题需要解决。

第一个问题是当一个测试用例失败时，我们调用`t.Fatalf`。这意味着在第一个失败的测试用例之后，我们停止测试其他用例。因为测试用例是按照无序的顺序运行的，所以如果有一个测试失败，那么如果我们可以知道它是唯一的失败还是第一个失败就很好了。

Golang的测试包会为我们做这件事，如果我们花费精力将每个测试用例写成自己的函数，但这是相当冗长的。好消息是，自从Go 1.7以来，一个新的特性被添加了进来，让我们可以很容易地为表驱动测试做到这一点。它们被称为[子测试](https://blog.golang.org/subtests)。

```go

func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) { //子集测试匿名函数
            got := Split(tc.input, tc.sep)
            if !reflect.DeepEqual(tc.want, got) {
                t.Fatalf("expected: %v, got: %v", tc.want, got)
            }
        })
    }
}

```

现在每一个子测试都有一个名字，我们可以在任何测试运行中自动打印出这个名字。

```bash
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]

```

每一个子测试都是它自己的匿名函数，因此我们可以使用`t.Fatalf`，`t.Skipf`和所有其他的`testing.T`helpers，同时保持表驱动测试的紧凑性。

### 每一个子测试都可以独立执行

因为子测试有一个名字，所以你可以使用`go test -run`标志来运行一个子测试的选择。

```bash
% go test -run=.*/trailing -v
=== RUN   TestSplit
=== RUN   TestSplit/trailing_sep
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]


```

### 比较测试的预期值(want)和实际值(got)


现在我们准备好修复测试用例了。让我们来看看错误。

```bash
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]

```

你可以看到问题吗？显然，这两个切片是不同的，这就是`reflect.DeepEqual`失败的原因。但是发现实际的差异并不容易，你必须注意到`c`后面的额外空格。这在这个简单的例子中看起来很简单，但是当你比较两个复杂的嵌套gRPC结构时，这是任何事情。


我们可以通过切换到`%#v`语法来改进输出，以将值视为Go(ish)声明：

```go
got := Split(tc.input, tc.sep)
if !reflect.DeepEqual(tc.want, got) {
    t.Fatalf("expected: %#v, got: %#v", tc.want, got)
}

```

现在当我们运行我们的测试时，我们可以轻易发现问题是切片中有一个额外的空元素 `""`。

```
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: []string{"a", "b", "c"}, got: []string{"a", "b", "c", ""}


```

但是在我们去修复我们的测试失败之前，我想再谈谈选择正确的方式来呈现测试失败。我们的`Split`函数很简单，它接受一个原始字符串并返回一个字符串切片，但是如果它使用结构体，或者更糟糕的是，指向结构体的指针呢？

这是个例子，`%#v` 当前并不适用：

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    fmt.Printf("%v %v\n", x, y)
    fmt.Printf("%#v %#v\n", x, y)
}

```

第一个`fmt.Printf`打印出了不太有用，但是预期的地址切片；`[0xc000096000 0xc000096008 0xc000096010] [0xc000096018 0xc000096020 0xc000096028]`。然而我们的`%#v`版本并没有更好，打印出了一个地址切片，转换为`*main.T`；`[]*main.T{(*main.T)(0xc000096000), (*main.T)(0xc000096008), (*main.T)(0xc000096010)} []*main.T{(*main.T)(0xc000096018), (*main.T)(0xc000096020), (*main.T)(0xc000096028)}`

现在因为使用`fmt.Printf`的限制，我想介绍一下来自Google的[go-cmp](https://github.com/google/go-cmp)


这个cmp库的目标是专门用来比较两个值。这类似于`reflect.DeepEqual`，但是它有更多的功能。使用cmp包，你当然可以写：

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    fmt.Println(cmp.Equal(x, y)) // false
}

```

但是对于我们的测试函数来说，更有用的是`cmp.Diff`函数，它将递归地产生两个值之间的差异的文本描述。

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    diff := cmp.Diff(x, y)
    fmt.Printf(diff)
}

```

上面的输出是：

```bash
% go run
{[]*main.T}[2].I:
         -: 3
         +: 4

```

它告诉我们，在`T`的切片的第2个元素中，`I`字段预期为3，但实际为4。

现在让我们把这些放在一起，使用go-cmp编写测试表格驱动测试

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            diff := cmp.Diff(tc.want, got)
            if diff != "" {
                t.Fatalf(diff)
            }
        })
    }
}

```
上面测试的输出是：

```bash
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:27: {[]string}[?->3]:
                -: <non-existent>
                +: ""
FAIL
exit status 1
FAIL    split   0.006s

```

我们可以看到，使用`cmp.Diff`不仅仅告诉我们得到的和我们想要的是不同的。同事还能看到字符串的长度不同，第三个索引在fixture中不应该存在，但是实际输出我们得到了一个空字符串，""。从这里修复测试失败是直截了当的。

