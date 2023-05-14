> 本文是 [Tutorial: Getting started with fuzzing](https://go.dev/doc/tutorial/fuzz)的中文翻译版本，内容有删减

本教程介绍了在Go中进行模糊测试的基础知识。使用模糊测试，会使用随机数据来运行测试，以查找漏洞或导致崩溃的输入。通过模糊测试可以发现的一些漏洞示例包括SQL注入、缓冲区溢出、拒绝服务和跨站点脚本攻击。

在本教程中，您将为一个简单函数编写一个模糊测试，运行go命令，并调试和修复代码中的问题。

在本教程中涉及到的术语可以参考[Go Fuzzing术语表](https://go.dev/security/fuzz/#glossary)

# 准备工作（Prerequisites）


*   **您必须安装了Go 1.18及以上版本.** 请参考 [Installing Go](https://go.dev/doc/install)来学习如何安装.
*   **代码编辑器.** 任何你有的文本编辑器都可以.
*   **命令行.** Go在Linux和Mac上使用任何终端都可以，也可以在Windows的PowerShell或cmd上使用.
*   **支持模糊测试的环境.** Go模糊测试仅在AMD64和ARM64架构上支持覆盖率插装.

#  创建文件夹

创建一个文件夹来存放你的代码

1.  打开命令行并切换到你的主目录
    
    Linux 或者 Mac上:
    ```bash
    $ cd
    ```
    Windows 上:
    ```
    C:\> cd %HOMEPATH%
    ```
    本文的剩余部分将显示$作为提示符。您使用的命令也可以在Windows上使用。
    
2.  在命令提示符下，创建一个名为fuzz的代码目录。
    
    ```bash
    $ mkdir fuzz
    $ cd fuzz
    
    ```
    
3.  创建一个模块来保存你的代码
    
    运行`go mod init`命令，给它你的新代码的模块路径。
    
    ```bash
    $ go mod init example/fuzz
    go: creating new go.mod: module example/fuzz
    ```
    
    **Note:** 对于生产代码，您将指定一个更符合自己需求的模块路径。更多信息，请参见[Managing dependencies](https://go.dev/doc/modules/managing-dependencies).
    

接下来，您将添加一些简单的代码来反转一个字符串，我们将在后面进行模糊测试。

# 添加测试代码

在这一步中，您将添加一个函数来反转一个字符串。

## 编写代码

1.  用你的文本编辑器，在fuzz目录下创建一个名为main.go的文件。
    
2.  在main.go的顶部，粘贴以下包声明。
    
    ```go
    package main
    ```
一个独立的程序（而不是一个库）总是在包`main`中。

3.  在包声明下面，粘贴以下函数声明。
    
    
    ```go
    func Reverse(s string) string {
        b := []byte(s)
        for i, j := 0, len(b)-1; i < len(b)/2; i, j = i+1, j-1 {
            b[i], b[j] = b[j], b[i]
        }
        return string(b)
    }
    ```
    
    这个函数的输入是一个`string`，然后将其转换为一个`byte`数组它，会逐个`byte`地遍历数组，最后返回一个反转的字符串。
    
    _Note:_ 这个代码是基于golang.org/x/example中的`stringutil.Reverse`函数。其他关于字符串拼接的介绍请参考[字符串拼接性能及原理](https://geektutu.com/post/hpg-string-concat.html)
    
4.  在main.go的顶部，在包声明下面，粘贴以下`main`函数来初始化一个字符串，反转它，打印输出，然后重复。
    
    ```go
    func main() {
        input := "The quick brown fox jumped over the lazy dog"
        rev := Reverse(input)
        doubleRev := Reverse(rev)
        fmt.Printf("original: %q\n", input)
        fmt.Printf("reversed: %q\n", rev)
        fmt.Printf("reversed again: %q\n", doubleRev)
    }
    ```
    
    这个函数将将运行`Reverse`，然后把输出打印到命令行。这对于查看代码的运行情况和调试可能有帮助。
    
5.   `main`函数也使用了fmt包，所以你需要导入它。
    
    程序的前几行代码应该是这样的:
    
    ```go
    package main
    
    import "fmt"
    
    ```
    

## 运行代码

在命令行中，进入包含main.go的目录，运行代码。


```bash
$ go run .
original: "The quick brown fox jumped over the lazy dog"
reversed: "god yzal eht revo depmuj xof nworb kciuq ehT"
reversed again: "The quick brown fox jumped over the lazy dog"

```

你可以看到原始字符串，反转后的结果，然后是再次反转后的结果，这与原始字符串相同。

现在代码已经运行了，是时候测试它了。

# 添加单元测试

在这一步中，您将为`Reverse`函数编写一个基本的单元测试。

## 编写代码

1.  使用你的文本编辑器，在fuzz目录下创建一个名为reverse_test.go的文件。
    
2.  粘贴以下代码到reverse_test.go中。
    
    ```go
    package main
    
    import (
        "testing"
    )
    
    func TestReverse(t *testing.T) {
        testcases := []struct {
            in, want string
        }{
            {"Hello, world", "dlrow ,olleH"},
            {" ", " "},
            {"!12345", "54321!"},
        }
        for _, tc := range testcases {
            rev := Reverse(tc.in)
            if rev != tc.want {
                    t.Errorf("Reverse: %q, want %q", rev, tc.want)
            }
        }
    }
    
    
    ```
    这个简单的测试将断言列出的输入字符串将被正确地反转。
    

## 运行代码

使用`go test`运行单元测试。

```go
$ go test
PASS
ok      example/fuzz  0.013s
```
接下来，您将把单元测试变成一个模糊测试。

# 添加一个模糊测试

单元测试有局限性，即每个输入都必须由开发人员添加到测试中。模糊测试的一个好处是，它会为您的代码提供输入，并可能识别出您提出的测试用例没有达到的边缘情况。


在本节中，您将把单元测试转换成模糊测试，这样您就可以用更少的工作量生成更多的输入了！


请注意，您可以将单元测试、基准测试和模糊测试保存在同一个*_test.go文件中，但是为了本例，您将把单元测试转换为模糊测试。

## 编写代码

在您的文本编辑器中，用以下模糊测试替换reverse_test.go中的单元测试。

```go
func FuzzReverse(f *testing.F) {
    testcases := []string{"Hello, world", " ", "!12345"}
    for _, tc := range testcases {
        f.Add(tc)  // Use f.Add to provide a seed corpus
    }
    f.Fuzz(func(t *testing.T, orig string) {
        rev := Reverse(orig)
        doubleRev := Reverse(rev)
        if orig != doubleRev {
            t.Errorf("Before: %q, after: %q", orig, doubleRev)
        }
        if utf8.ValidString(orig) && !utf8.ValidString(rev) {
            t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
        }
    })
}
```

模糊测试也有一些局限性。在单元测试中，您可以预测`Reverse`函数的预期输出，并验证实际输出是否符合预期。

For example, in the test case `Reverse("Hello, world")` the unit test specifies the return as `"dlrow ,olleH"`.

当使用模糊测试时，您无法预测预期的输出，因为您无法控制输入。


然而，有一些`Reverse`函数的属性是您可以在模糊测试中验证的。在这个模糊测试中要检查的两个属性是:

1.  反转两次后，字符串的值保持不变

2. 反转的字符串保持其状态为有效的UTF-8。

请注意单元测试和模糊测试之间的语法差异:

*  模糊测试以FuzzXxx开头，而不是TestXxx，并且以`*testing.F`作为输入，而不是`*testing.T`。
*  在你期望看到一个`t.Run`执行的地方，你会看到`f.Fuzz`，它接受一个模糊目标函数，它的参数是`*testing.T`和要模糊的类型。使用`f.Add`提供单元测试中的输入作为种子语料库输入。
确保已经导入了新的包`unicode/utf8`。

```go
package main

import (
    "testing"
    "unicode/utf8"
)
```
将单元测试转换为模糊测试后，现在是再次运行测试的时候了。

## 运行代码

1.  首先以不模糊的方式运行测试，以确保种子输入通过。
    
    ```bash
    $ go test
    PASS
    ok      example/fuzz  0.013s
    
    ```
    你也可以运行命令`go test -run=FuzzReverse`，如果你在那个文件中有其他的测试，并且你只想运行模糊测试。
    
2.  运行`FuzzReverse`进行模糊测试，看看是否有任何随机生成的字符串输入会导致失败。这是使用`go test`执行的，其中一个新的标志`-fuzz`设置为参数`Fuzz`。请运行下面的命令。
    
    ```
    $ go test -fuzz=Fuzz
    ```
    
    
      另一个有用的标志是`-fuzztime`，它限制了模糊测试所花费的时间。例如，在下面的测试中指定`-fuzztime 10s`意味着，只要之前没有发生故障，测试就会在默认情况下在10秒内退出。请参阅cmd/go文档的[本节](https://pkg.go.dev/cmd/go#hdr-Testing_flags)以查看其他测试标志。


    现在，运行你刚刚复制的命令。

    
    ```bash
    $ go test -fuzz=Fuzz
    fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
    fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 8 workers
    fuzz: minimizing 38-byte failing input file...
    --- FAIL: FuzzReverse (0.01s)
        --- FAIL: FuzzReverse (0.00s)
            reverse_test.go:20: Reverse produced invalid UTF-8 string "\x9c\xdd"
    
        Failing input written to testdata/fuzz/FuzzReverse/af69258a12129d6cbba438df5d5f25ba0ec050461c116f777e77ea7c9a0d217a
        To re-run:
        go test -run=FuzzReverse/af69258a12129d6cbba438df5d5f25ba0ec050461c116f777e77ea7c9a0d217a
    FAIL
    exit status 1
    FAIL    example/fuzz  0.030s
    ```
    一个失败发生在模糊测试中，导致问题的输入被写入一个种子语料库文件，下次调用`go test`时，即使没有`-fuzz`标志，也会运行该文件。要查看导致失败的输入，请在文本编辑器中打开testdata/fuzz/FuzzReverse目录中的语料库文件。您的种子语料库文件可能包含一个不同的字符串，但格式是相同的。

    
    ```bash
    go test fuzz v1
    string("泃")
    ```
    
    语料库文件的第一行表示编码版本。每个后续行表示构成语料库条目的每个类型的值。由于模糊目标只需要1个输入，因此在版本之后只有1个值。
    

    
3.  运行`go test`，不带`-fuzz`标志;将使用新的失败种子语料库条目:
    
    ```bash
    $ go test
    --- FAIL: FuzzReverse (0.00s)
        --- FAIL: FuzzReverse/af69258a12129d6cbba438df5d5f25ba0ec050461c116f777e77ea7c9a0d217a (0.00s)
            reverse_test.go:20: Reverse produced invalid string
    FAIL
    exit status 1
    FAIL    example/fuzz  0.016s
    
    
    ```
    
    由于我们的测试失败了，现在是解决问题的时候了。
    

# 修复无效字符串错误

在本节中，您将调试失败，并修复错误。

请随意花一些时间思考这个问题，并在继续之前尝试修复这个问题。

## 定位错误
 有几种不同的方法可以调试这个错误。如果您使用VS Code作为文本编辑器，您可以参考[set up your debugger](https://github.com/golang/vscode-go/blob/master/docs/debugging.md)


在这个教程中，我们将把有用的调试信息记录到你的终端。这些信息将帮助您定位错误。

首先，考虑[`utf8.ValidString`](https://pkg.go.dev/unicode/utf8)的文档。

```
ValidString reports whether s consists entirely of valid UTF-8-encoded runes.
```

当前的`Reverse`函数逐字节地反转字符串，其中就有我们的问题所在。为了保留原始字符串的UTF-8编码的符文，我们必须逐符文地反转字符串。

为了检查为什么输入(在这种情况下，中文字符`泃`)导致`Reverse`在反转时产生一个无效的字符串，您可以检查反转字符串中的符文数。

### 编写代码

在文本编辑器中，用以下内容替换`FuzzReverse`中的模糊目标。

```go
f.Fuzz(func(t *testing.T, orig string) {
    rev := Reverse(orig)
    doubleRev := Reverse(rev)
    t.Logf("Number of runes: orig=%d, rev=%d, doubleRev=%d", utf8.RuneCountInString(orig), utf8.RuneCountInString(rev), utf8.RuneCountInString(doubleRev))
    if orig != doubleRev {
        t.Errorf("Before: %q, after: %q", orig, doubleRev)
    }
    if utf8.ValidString(orig) && !utf8.ValidString(rev) {
        t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
    }
})


```

这个`t.Logf`行将在发生错误时打印到命令行，或者在使用`-v`执行测试时，这可以帮助您调试这个特定的问题。

### 运行代码

运行 `go test`

```bash
$ go test
--- FAIL: FuzzReverse (0.00s)
    --- FAIL: FuzzReverse/28f36ef487f23e6c7a81ebdaa9feffe2f2b02b4cddaa6252e87f69863046a5e0 (0.00s)
        reverse_test.go:16: Number of runes: orig=1, rev=3, doubleRev=1
        reverse_test.go:21: Reverse produced invalid UTF-8 string "\x83\xb3\xe6"
FAIL
exit status 1
FAIL    example/fuzz    0.598s


```

整个种子语料库使用的字符串中，每个字符都是一个单字节。但是，像`泃`这样的字符可能需要多个字节。因此，逐字节颠倒字符串将使多字节字符无效。

**Note:** 如果你对 Go 如何处理字符串，请参考[Strings, bytes, runes and characters in Go](https://go.dev/blog/strings) 来深入理解

在更好地理解错误之后，我们纠正`Reverse`函数中的错误。

## 修复错误

为了纠正`Reverse`函数，让我们通过符文遍历字符串，而不是通过字节。

### 编写代码

在你的文本编辑器中，用以下内容替换现有的`Reverse()`函数。

```go
func Reverse(s string) string {
    r := []rune(s)
    for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
        r[i], r[j] = r[j], r[i]
    }
    return string(r)
}


```
这里的关键区别是`Reverse`现在是在字符串中迭代每个`rune`，而不是每个`byte`。

### 运行代码

1.  运行 `go test`
    
    ```bash
    $ go test
    PASS
    ok      example/fuzz  0.016s
    
    
    ```
    
    新的测试通过了！
    
2.  使用`go test -fuzz`再次运行模糊测试，看看是否有新的错误。
    
    ```bash
    $ go test -fuzz=Fuzz
    fuzz: elapsed: 0s, gathering baseline coverage: 0/37 completed
    fuzz: minimizing 506-byte failing input file...
    fuzz: elapsed: 0s, gathering baseline coverage: 5/37 completed
    --- FAIL: FuzzReverse (0.02s)
        --- FAIL: FuzzReverse (0.00s)
            reverse_test.go:33: Before: "\x91", after: "�"
    
        Failing input written to testdata/fuzz/FuzzReverse/1ffc28f7538e29d79fce69fef20ce5ea72648529a9ca10bea392bcff28cd015c
        To re-run:
        go test -run=FuzzReverse/1ffc28f7538e29d79fce69fef20ce5ea72648529a9ca10bea392bcff28cd015c
    FAIL
    exit status 1
    FAIL    example/fuzz  0.032s
    
    
    ```
    我们可以看到，在颠倒两次后，该字符串与原始字符串不同。这次输入本身就是无效的Unicode。如果我们使用字符串进行模糊测试，这是怎么可能的？

    

# 修复双重反转错误
在这一节中，您将调试双重反转失败并修复错误。

## 定位错误

像之前一样，您可以使用多种方法调试此失败。在这种情况下，使用[debugger](https://github.com/golang/vscode-go/blob/master/docs/debugging.md)将会是一个很好的方法。

在这个教程中，我们将在`Reverse`函数中记录有用的调试信息。


仔细查看反转的字符串以发现错误。在 Go 中，[字符串是只读的字节切片](https://go.dev/blog/strings)，可以包含不是有效 UTF-8 的字节。原始字符串是一个具有一个字节的字节切片，为`'\x91'`。当输入字符串设置为`[]rune`时，Go 将字节切片编码为 UTF-8，并用 UTF-8 字符�替换字节。当我们将替换的 UTF-8 字符与输入字节切片进行比较时，它们显然不相等。



### 编写代码

1.  在你的文本编辑器中，用以下内容替换现有的`Reverse()`函数。
    
    ```go

    func Reverse(s string) string {
        fmt.Printf("input: %q\n", s)
        r := []rune(s)
        fmt.Printf("runes: %q\n", r)
        for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
            r[i], r[j] = r[j], r[i]
        }
        return string(r)
    }
    
    
    ```
    这将帮助我们理解将字符串转换为符文切片时出了什么问题。
    

### 运行代码

这次，我们只想运行失败的测试，以便检查日志。为此，我们将使用`go test -run`。

为了运行 FuzzXxx/testdata 中的特定语料库条目，您可以在`-run`中提供`{FuzzTestName}/{filename}`。这在调试时可能很有用。在这种情况下，将`-run`标志设置为失败测试的确切哈希值。从终端复制并粘贴唯一的哈希值；它将与下面的哈希值不同。

```bash
$ go test -run=FuzzReverse/28f36ef487f23e6c7a81ebdaa9feffe2f2b02b4cddaa6252e87f69863046a5e0
input: "\x91"
runes: ['�']
input: "�"
runes: ['�']
--- FAIL: FuzzReverse (0.00s)
    --- FAIL: FuzzReverse/28f36ef487f23e6c7a81ebdaa9feffe2f2b02b4cddaa6252e87f69863046a5e0 (0.00s)
        reverse_test.go:16: Number of runes: orig=1, rev=1, doubleRev=1
        reverse_test.go:18: Before: "\x91", after: "�"
FAIL
exit status 1
FAIL    example/fuzz    0.145s


```
 我们知道输入是无效的Unicode，让我们在`Reverse`函数中修复错误。

## 修复错误

 为了修复这个问题，让我们在`Reverse`的输入不是有效的UTF-8时返回一个错误。

### 编写代码

1.  在你的文本编辑器中，用以下内容替换现有的`Reverse()`函数。
    
    ```go
    func Reverse(s string) (string, error) {
        if !utf8.ValidString(s) {
            return s, errors.New("input is not valid UTF-8")
        }
        r := []rune(s)
        for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
            r[i], r[j] = r[j], r[i]
        }
        return string(r), nil
    }
    
    
    ```
    本次更改将在输入字符串包含无效的UTF-8字符时返回错误。
    
2.  由于`Reverse`函数现在返回一个错误，所以修改`main`函数以丢弃额外的错误值。用以下内容替换现有的`main`函数。
    
    ```go
    func main() {
        input := "The quick brown fox jumped over the lazy dog"
        rev, revErr := Reverse(input)
        doubleRev, doubleRevErr := Reverse(rev)
        fmt.Printf("original: %q\n", input)
        fmt.Printf("reversed: %q, err: %v\n", rev, revErr)
        fmt.Printf("reversed again: %q, err: %v\n", doubleRev, doubleRevErr)
    }
    ```
    这些对`Reverse`的调用应该返回一个 nil，因为输入字符串是有效的UTF-8。
    
    
3.   你需要导入errors和unicode/utf8包。main.go中的导入语句应该如下所
   
    ```
    import (
        "errors"
        "fmt"
        "unicode/utf8"
    )
    ```
    
4.   修改reverse_test.go文件以检查错误并跳过测试，如果错误由返回生成。
    ```
    func FuzzReverse(f *testing.F) {
        testcases := []string {"Hello, world", " ", "!12345"}
        for _, tc := range testcases {
            f.Add(tc)  // Use f.Add to provide a seed corpus
        }
        f.Fuzz(func(t *testing.T, orig string) {
            rev, err1 := Reverse(orig)
            if err1 != nil {
                return
            }
            doubleRev, err2 := Reverse(rev)
            if err2 != nil {
                 return
            }
            if orig != doubleRev {
                t.Errorf("Before: %q, after: %q", orig, doubleRev)
            }
            if utf8.ValidString(orig) && !utf8.ValidString(rev) {
                t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
            }
        })
    }
    
    
    ```
    
    Rather than returning, you can also call `t.Skip()` to stop the execution of that fuzz input.
    

### 运行代码

运行 `go test`
    
    ```
    $ go test
    PASS
    ok      example/fuzz  0.019s
    ```
    
使用`go test -fuzz=Fuzz`运行模糊测试，然后在几秒钟后停止模糊测试`ctrl-C`。如果没有发生故障，默认情况下，模糊测试将一直运行，进程可以使用`ctrl-C`中断。
    

```bash
$ go test -fuzz=Fuzz
fuzz: elapsed: 0s, gathering baseline coverage: 0/38 completed
fuzz: elapsed: 0s, gathering baseline coverage: 38/38 completed, now fuzzing with 4 workers
fuzz: elapsed: 3s, execs: 86342 (28778/sec), new interesting: 2 (total: 35)
fuzz: elapsed: 6s, execs: 193490 (35714/sec), new interesting: 4 (total: 37)
fuzz: elapsed: 9s, execs: 304390 (36961/sec), new interesting: 4 (total: 37)
...
fuzz: elapsed: 3m45s, execs: 7246222 (32357/sec), new interesting: 8 (total: 41)
^Cfuzz: elapsed: 3m48s, execs: 7335316 (31648/sec), new interesting: 8 (total: 41)
PASS
ok      example/fuzz  228.000s


```

运行`go test -fuzz=Fuzz -fuzztime 30s`，如果没有发现故障，将在30秒内模糊测试后退出。
    
    ```bash
    $ go test -fuzz=Fuzz -fuzztime 30s
    fuzz: elapsed: 0s, gathering baseline coverage: 0/5 completed
    fuzz: elapsed: 0s, gathering baseline coverage: 5/5 completed, now fuzzing with 4 workers
    fuzz: elapsed: 3s, execs: 80290 (26763/sec), new interesting: 12 (total: 12)
    fuzz: elapsed: 6s, execs: 210803 (43501/sec), new interesting: 14 (total: 14)
    fuzz: elapsed: 9s, execs: 292882 (27360/sec), new interesting: 14 (total: 14)
    fuzz: elapsed: 12s, execs: 371872 (26329/sec), new interesting: 14 (total: 14)
    fuzz: elapsed: 15s, execs: 517169 (48433/sec), new interesting: 15 (total: 15)
    fuzz: elapsed: 18s, execs: 663276 (48699/sec), new interesting: 15 (total: 15)
    fuzz: elapsed: 21s, execs: 771698 (36143/sec), new interesting: 15 (total: 15)
    fuzz: elapsed: 24s, execs: 924768 (50990/sec), new interesting: 16 (total: 16)
    fuzz: elapsed: 27s, execs: 1082025 (52427/sec), new interesting: 17 (total: 17)
    fuzz: elapsed: 30s, execs: 1172817 (30281/sec), new interesting: 17 (total: 17)
    fuzz: elapsed: 31s, execs: 1172817 (0/sec), new interesting: 17 (total: 17)
    PASS
    ok      example/fuzz  31.025s
    
    
    ```
    
现在模糊测试通过！

除了 `-fuzz`标志外，还向`go test`添加了几个新标志，可以在文档中查看[documentation](https://go.dev/security/fuzz/#custom-settings)
    
有关模糊测试输出中使用的术语，请参考[Go Fuzzing](https://go.dev/security/fuzz/#command-line-output)
    

# 总结


接下来的步骤是选择要模糊的代码中的一个函数，然后尝试一下！如果模糊测试发现了代码中的错误，请考虑将其添加到[trophy case](https://github.com/golang/go/wiki/Fuzzing-trophy-case)
 如果您遇到任何问题或有功能想法，请[提交问题](https://github.com/golang/go/issues/new/?&labels=fuzz)

参考[go.dev/security/fuzz](https://go.dev/security/fuzz/#requirements)的文档进行进一步阅读。

# 完整代码



``` go
// — main.go —
package main

import (
    "errors"
    "fmt"
    "unicode/utf8"
)

func main() {
    input := "The quick brown fox jumped over the lazy dog"
    rev, revErr := Reverse(input)
    doubleRev, doubleRevErr := Reverse(rev)
    fmt.Printf("original: %q\n", input)
    fmt.Printf("reversed: %q, err: %v\n", rev, revErr)
    fmt.Printf("reversed again: %q, err: %v\n", doubleRev, doubleRevErr)
}

func Reverse(s string) (string, error) {
    if !utf8.ValidString(s) {
        return s, errors.New("input is not valid UTF-8")
    }
    r := []rune(s)
    for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
        r[i], r[j] = r[j], r[i]
    }
    return string(r), nil
}


```



```go
// — reverse_test.go —
package main

import (
    "testing"
    "unicode/utf8"
)

func FuzzReverse(f *testing.F) {
    testcases := []string{"Hello, world", " ", "!12345"}
    for _, tc := range testcases {
        f.Add(tc) // Use f.Add to provide a seed corpus
    }
    f.Fuzz(func(t *testing.T, orig string) {
        rev, err1 := Reverse(orig)
        if err1 != nil {
            return
        }
        doubleRev, err2 := Reverse(rev)
        if err2 != nil {
            return
        }
        if orig != doubleRev {
            t.Errorf("Before: %q, after: %q", orig, doubleRev)
        }
        if utf8.ValidString(orig) && !utf8.ValidString(rev) {
            t.Errorf("Reverse produced invalid UTF-8 string %q", rev)
        }
    })
}


```
