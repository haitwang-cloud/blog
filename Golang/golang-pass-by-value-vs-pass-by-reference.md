> 本文是[Golang Pass by value vs Pass by reference](https://david-yappeter.medium.com/golang-pass-by-value-vs-pass-by-reference-e48aac8b2716)的中文翻译版本，内容有删减


![](./pics/golang-pass-by.gif)

**值传递**和**引用传递**是使用指针类型时需要尤其注意的点（特别是在Java, C#, C/C++和Go等语言中）

当使创建方法/函数包含参数时，该参数可以选择使用普通数据类型或指针来进行传递。这将使传递给方法的参数有所不同：
- **按值传递** 会将变量的值传递到方法中，或者可以说是将原始变量将值“复制”到另一个内存位置并将新创建的值传递到方法中。因此，在方法中变量发生的任何改变都不会影响原始变量值。
- **按引用传递** 将传递变量的内存位置而不是值。换句话说，它将变量的“容器”传递给方法，因此，方法中变量的任何改变都会影响原始变量。


> 简而言之,按值传递将复制值，按引用传递将传递内存位置。


下图是清晰得解释了**按值传递**和**按引用传递**的区别

![][img-1]source: [www.penjee.com](http://www.penjee.com/)

在Go语言中，我们可以处理指针，因此我们必须了解按值传递和按引用传递在 Go 中的工作原理，本文将它分为 3 个部分：“基本”、“引用”、“函数”。

源代码地址: [https://github.com/david-yappeter/go-passbyvalue-passbyreference](https://github.com/david-yappeter/go-passbyvalue-passbyreference)

### 基本数据类型(Basic Data Type）
-------------------

[basicAndArray](https://github.com/david-yappeter/go-passbyvalue-passbyreference/blob/master/01.basicAndArray/main.go)


```Golang
package main

import "fmt"

func main() {
	a, b := 0, 0

	// Initialize Value
	fmt.Printf("## INIT\n")
	fmt.Printf("Memory Location a: %p, b: %p\n", &a, &b)
	fmt.Printf("Value a: %d, b: %d\n", a, b) // 0 0

	// Passing By Value a(int)
	Add(a) // Golang will copy value of 'a' and insert it into argument

	// Passing By Reference b(int), &b(*int) => with '&' we can get the memory location of 'b'
	AddPtr(&b) // Pass Memory Location of 'b' into argument

	fmt.Printf("\n## FINAL\n")
	fmt.Printf("Memory Location a: %p, b: %p\n", &a, &b)
	fmt.Printf("Value a: %d, b: %d\n", a, b) // 0 1
}

// Pass By Value
func Add(x int) {
	fmt.Printf("\n## 'Add' Function\n")
	fmt.Printf("Before Add, Memory Location: %p, Value: %d\n", &x, x)
	x++
	fmt.Printf("After Add, Memory Location: %p, Value: %d\n", &x, x)
}

// Pass By Reference
func AddPtr(x *int) {
	fmt.Printf("\n## 'AddPtr' Function\n")
	fmt.Printf("Before AddPtr, Memory Location: %p, Value: %d\n", x, *x)
	*x++ // We add * in front of the variable because it is a pointer, * will call value of a pointer
	fmt.Printf("After AddPtr, Memory Location: %p, Value: %d\n", x, *x)
}
```

Output:

```Golang
## INIT
Memory Location a: 0xc0000120a8, b: 0xc0000120c0      
Value a: 0, b: 0

## 'Add' Function
Before Add, Memory Location: 0xc0000120f0, Value: 0   
After Add, Memory Location: 0xc0000120f0, Value: 1

## 'AddPtr' Function
Before AddPtr, Memory Location: 0xc0000120c0, Value: 0
After AddPtr, Memory Location: 0xc0000120c0, Value: 1

## FINAL
Memory Location a: 0xc0000120a8, b: 0xc0000120c0      
Value a: 0, b: 1

```
在第一个例子中,我们将研究基本数据类型。以下有 2 个函数：
- `Add(x int)` 的输入是一个integer
- `AddPtr(x *int)`的输入是一个integer指针

`a`和`b`这两个变量在内存中的位置无法预测，但我们可以通过打印它们的内存位置来追踪它们。

在`Add`函数中，`a`在`main()`函数中的位置与在`Add`不通，是因为 Go 复制 `a`的值并在内存的其他位置重新初始化一个地址，所以如果我们通过`x++`是不能更改`main()`函数中的`a`的值。最终因为是**按值传递**`a`的值还是0


在`AddPtr`函数中，`b`在`main()`函数中的内存位置与在`AddPtr`是一致的，我们在`AddPtr`函数对`x` 的操作最终会影响 `b` 的值。因此当我们尝试通过`*x++`来更改`x`值的时候，最终因为当前是**按引用传递**`b`的值会变成1。


其他的基本数据类型比如`int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, uintptr, float32, float64, string, bool, byte, rune, Array, Structs`都符合上述规律

### 引用数据类型(Referenced Data Type）
------------------------

[referenced](https://github.com/david-yappeter/go-passbyvalue-passbyreference/blob/master/02.referenced/main.go)

```Golang
package main

import (
	"fmt"
)

func main() {
	// Slices
	fmt.Println("======================")
	fmt.Println("SLICES")
	fmt.Println("======================")
	var arrInt []int = []int{1, 2, 3, 4, 5}
	var sliceInt = arrInt[3:]

	fmt.Printf("Init\n")
	fmt.Printf("ArrInt: %+v, SliceInt: %+v\n\n", arrInt, sliceInt)

	sliceInt[0] = 10
	fmt.Printf("After\n")
	fmt.Printf("ArrInt: %+v, SliceInt: %+v\n", arrInt, sliceInt)

	// MAP
	fmt.Println("======================")
	fmt.Println("MAP")
	fmt.Println("======================")
	var emptyMap = make(map[string]interface{})
	fmt.Println("Init")
	fmt.Printf("emptyMap : %+v\n", emptyMap)

	MapFunc(emptyMap)
	fmt.Println("After")
	fmt.Printf("emptyMap : %+v\n", emptyMap)
}

func MapFunc(val map[string]interface{}) {
	val["this is a new value"] = 100
}
```

Output:

```Golang
======================
SLICES
======================
Init
ArrInt: [1 2 3 4 5], SliceInt: [4 5]

After
ArrInt: [1 2 3 10 5], SliceInt: [10 5] 
======================
MAP
======================
Init
emptyMap : map[]

After
emptyMap : map[this is a new value:100]

```

在第二个例子中，数组中的`Slices`因为和数组共享内存位置，因此如果我们更改`Slices`的值，它将影响数组值，并且`map`数据类型默认为按引用传递，因此函数内部的任何更改都将更改原始值。如果我们想为`map`做一个传递值，我们可以使用[`copy`](https://gosamples.dev/copy-map/)，它将创建`map`的新变量，这样函数中值的改变不会影响原始值。[`chan`](https://go.dev/tour/concurrency/2)或通道也是引用的数据类型。

### 函数(Function）
------------
```Golang
package main

import "fmt"

type StructVal struct {
	IntVal int
}

func (s StructVal) Add() {
	fmt.Printf("====================\n")
	fmt.Printf("Func Add\n")
	fmt.Printf("====================\n")
	fmt.Printf("Memory Location %p\n", &s)
	fmt.Printf("Value Before: %+v\n", s)
	s.IntVal++
	fmt.Printf("Value After: %+v\n\n", s)
}

func (s *StructVal) AddPtr() {
	fmt.Printf("====================\n")
	fmt.Printf("Func AddPtr\n")
	fmt.Printf("====================\n")
	fmt.Printf("Memory Location %p\n", s)
	fmt.Printf("Value Before: %+v\n", s)
	s.IntVal++
	fmt.Printf("Value After: %+v\n\n", s)
}

func main() {
	init := StructVal{
		IntVal: 0,
	}

	fmt.Printf("================\n")
	fmt.Printf("MAIN\n")
	fmt.Printf("================\n")
	fmt.Printf("Memory Location: %p\n", &init)
	fmt.Printf("Value: %+v\n\n", init)

	init.Add()

	fmt.Printf("================\n")
	fmt.Printf("AFTER Func Add()\n")
	fmt.Printf("================\n")
	fmt.Printf("Value: %+v\n\n", init)

	init.AddPtr()
	fmt.Printf("================\n")
	fmt.Printf("AFTER Func AddPtr()\n")
	fmt.Printf("================\n")
	fmt.Printf("Value: %+v\n\n", init)

	fmt.Printf("================\n")
	fmt.Printf("FINAL\n")
	fmt.Printf("================\n")
	fmt.Printf("Value: %+v\n", init)
}
```
Output:

```Golang
================
MAIN
================
Memory Location: 0xc00001c030
Value: {IntVal:0}

====================
Func Add
====================
Memory Location 0xc00001c040
Value Before: {IntVal:0}
Value After: {IntVal:1}

================
AFTER Func Add()
================
Value: {IntVal:0}

====================
Func AddPtr
====================
Memory Location 0xc00001c030
Value Before: &{IntVal:0}
Value After: &{IntVal:1}

================
AFTER Func AddPtr()
================
Value: {IntVal:1}

================
FINAL
================
Value: {IntVal:1}

```

在最后一个例子中，我们将处理 `Struct Function`。不同之处在于`Struct`本身，而不是`Struct`内部的变量。当我们创建一个`Struct`函数时，我们将`Struct`声明为函数的前缀，例如`func (s struct) FuncName()`或`func (s *struct) FuncName()`。不同之处在于，如果我们使用指针类型的函数`func (s *struct) FuncName()`，它的输入是原始的`Struct`，因此`Struct`值发生的任何变化都会对原始`Struct`产生影响，而没有指针的函数`func (s struct) FuncName()` ，则将复制`Struct`并将其传递给函数。  因此，如果`Struct`的值在函数中发生任何变化，它不会影响原始结构。

