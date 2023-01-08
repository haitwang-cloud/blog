> [3 ways to compare slices (arrays)](https://yourbasic.org/golang/compare-slices/)的中文翻译版本，添加了benchMark 测试的结果



### 基本案例(Basic case）
----------

在大多数情况下，你需要自己实现比较两个 [**slices**](https://yourbasic.org/golang/slices-explained/) 的代码

```Golang
// Equal tells whether a and b contain the same elements.
// A nil argument is equivalent to an empty slice.
func Equal(a, b []int) bool {
    if len(a) != len(b) {
        return false
    }
    for i, v := range a {
        if v != b[i] {
            return false
        }
    }
    return true
}
```

对于  [**arrays**](https://yourbasic.org/golang/slices-explained/)来说，你可以使用  `==` 或者`!=`

```
a := [2]int{1, 2}
b := [2]int{1, 3}
fmt.Println(a == b) // false
```

>只有在数组的元素类型的值是可比较的时候，才可以使用 `==`和`!=`。如果它们的对应元素相等，则两个数组值相等。
[The Go Programming Language Specification: Comparison operators](https://golang.org/ref/spec#Comparison_operators)

### 对slices优化的代码(Optimized code for byte slices）
------------------------------

如果是为了比较byte slices，可以使用优化过的方法 [`bytes.Equal`](https://golang.org/pkg/bytes/#Equal)。此函数会将 nil 视为等效于空切片。

### 用于递归比较的通用代码(General-purpose code for recursive comparison)
---------------------------------------------

当在测试的时候，您可能需要使用 [`reflect.DeepEqual`](https://golang.org/pkg/reflect/#DeepEqual)，它会递归地比较任何类型的两个元素。

```
var a []int = nil
var b []int = make([]int, 0)
fmt.Println(reflect.DeepEqual(a, b)) // false
```

这个函数的性能比上面的代码差很多，但它在简单性和正确性至关重要的测试用例中很有用。然而，这样做的代码语法会变得相当复杂。

BenchMark 测试
----------

**笔者为了验证两种方法equal的性能，做了下面的benchMark测试，可以看到，使用reflect.DeepEqual的确实性能比较差，而使用自己的方法的性能比较好。**
```
(base)  ~/go/src/goTest/benchmark/ go test  -bench=. -benchtime=5s -count=2 -cpu=2,4,8,16
goos: darwin
goarch: amd64
pkg: goTest/benchmark
cpu: Intel(R) Core(TM) i9-9980HK CPU @ 2.40GHz
BenchmarkEqual-2                1000000000               4.910 ns/op
BenchmarkEqual-2                1000000000               4.866 ns/op
BenchmarkEqual-4                1000000000               4.912 ns/op
BenchmarkEqual-4                1000000000               4.913 ns/op
BenchmarkEqual-8                1000000000               4.833 ns/op
BenchmarkEqual-8                1000000000               4.894 ns/op
BenchmarkEqual-16               1000000000               4.834 ns/op
BenchmarkEqual-16               1000000000               4.925 ns/op
BenchmarkDeepEqual-2             6886509               849.3 ns/op
BenchmarkDeepEqual-2             7062068               854.2 ns/op
BenchmarkDeepEqual-4             7145196               858.4 ns/op
BenchmarkDeepEqual-4             7132903               847.5 ns/op
BenchmarkDeepEqual-8             7086793               908.5 ns/op
BenchmarkDeepEqual-8             6742137               858.6 ns/op
BenchmarkDeepEqual-16            6991735               863.7 ns/op
BenchmarkDeepEqual-16            7088943               849.6 ns/op
PASS
ok      goTest/benchmark        98.647s

```

Source code
------------------------------
 `euqal.go`
```
package benchmark

import "reflect"

var (
	target = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	source = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
)

func Equal() bool {
	if len(source) != len(target) {
		return false
	}
	for i, v := range source {
		if v != target[i] {
			return false
		}
	}
	return true
}

func DeepEqual() bool {
	return reflect.DeepEqual(source, target)

}

```
 `euqal_test.go`
```
package benchmark

import (
	"testing"
)

func BenchmarkEqual(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Equal()
	}
}

func BenchmarkDeepEqual(b *testing.B) {
	for i := 0; i < b.N; i++ {
		DeepEqual()
	}
}

```