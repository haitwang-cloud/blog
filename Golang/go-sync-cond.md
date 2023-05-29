> 本文是 [How to properly use the conditional variable sync.Cond in Golang](https://www.sobyte.net/post/2022-07/go-sync-cond/)的中文翻译版本，内容有删减


Golang sync包中的Cond实现了一个条件变量，可以用在多个Reader等待一个共享资源ready的场景中（如果只有一个读一个写，此时锁或者channel就可以搞定）。

Cond pool的点在于：多个goroutine等待，1个goroutine发生事件通知。


每一个Cond都关联了一个Lock(`*sync.Mutex or *sync.RWMutex`)，在修改条件或者调用Wait方法时必须加锁，**保护条件**。

```go
type Cond struct {
        // L is held while observing or changing the condition
        L Locker
        // contains filtered or unexported fields
}
```
创建一个新的Cond条件变量。
```go
func NewCond(l Locker) *Cond
```



Broadcast 会唤醒所有等待的goroutine。

同时，Broadcast也可以不加锁调用。

```go
func (c *Cond) Broadcast()
```

Signal只会唤醒一个等待的goroutine。

```go
func (c *Cond) Signal()
```

Signal可以不加锁调用，但是如果不加锁调用，那么Signal必须在Wait之前调用，否则会panic。

`Wait()` 会自动释放 `c.L` 并挂起调用者的goroutine，`Wait()` 返回时会对 `c.L` 上锁。

`Wait()` 不会主动return，除非它被Signal或者Broadcast唤醒。

由于 `C.L` 在 `Wait()` 第一次恢复时没有锁定，因此调用者通常不应认为 `Wait()` 返回时条件为真。

调用者应该在循环中调用Wait。（简单来说，每当你想使用条件时，都必须加锁。）

```go
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```

下面的例子更好的说明了Cond的使用。

[Playground1](https://play.golang.org/p/g2hyc2yDdJu)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var sharedRsc = false

func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    m := sync.Mutex{}
    c := sync.NewCond(&m)
    go func() {
        // this go routine wait for changes to the sharedRsc
        c.L.Lock()
        for sharedRsc == false {
            fmt.Println("goroutine1 wait")
            c.Wait()
        }
        fmt.Println("goroutine1", sharedRsc)
        c.L.Unlock()
        wg.Done()
    }()

    go func() {
        // this go routine wait for changes to the sharedRsc
        c.L.Lock()
        for sharedRsc == false {
            fmt.Println("goroutine2 wait")
            c.Wait()
        }
        fmt.Println("goroutine2", sharedRsc)
        c.L.Unlock()
        wg.Done()
    }()

    // this one writes changes to sharedRsc
    time.Sleep(2 * time.Second)
    c.L.Lock()
    fmt.Println("main goroutine ready")
    sharedRsc = true
    c.Broadcast()
    fmt.Println("main goroutine broadcast")
    c.L.Unlock()
    wg.Wait()
}
```

程序执行结果如下。

```
goroutine1 wait
goroutine2 wait
main goroutine ready
main goroutine broadcast
goroutine2 true
goroutine1 true
```


goroutine1 和 goroutine2 进入 Wait 状态，等待 main goroutine 在2s后资源满足后，发出 broadcast 信号，从 Wait 中恢复，判断条件是否确实满足（sharedRsc不为空），满足则消费条件并解锁，`wg.Done()`。

接下来，我们对main goroutine中的2s延迟进行了修改。


```go
package main

import (
	"fmt"
	"sync"
)

var sharedRsc = false

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	m := sync.Mutex{}
	c := sync.NewCond(&m)
	go func() {
		// this go routine wait for changes to the sharedRsc
		c.L.Lock()
		for sharedRsc == false {
			fmt.Println("goroutine1 wait")
			c.Wait()
		}
		fmt.Println("goroutine1", sharedRsc)
		c.L.Unlock()
		wg.Done()
	}()

	go func() {
		// this go routine wait for changes to the sharedRsc
		c.L.Lock()
		for sharedRsc == false {
			fmt.Println("goroutine2 wait")
			c.Wait()
		}
		fmt.Println("goroutine2", sharedRsc)
		c.L.Unlock()
		wg.Done()
	}()

	// this one writes changes to sharedRsc
	c.L.Lock()
	fmt.Println("main goroutine ready")
	sharedRsc = true
	c.Broadcast()
	fmt.Println("main goroutine broadcast")
	c.L.Unlock()
	wg.Wait()
}
```

The execution result is as follows.

程序执行结果如下    

```bash
main goroutine ready
main goroutine broadcast
goroutine2 true
goroutine1 true
```


有趣的是，两个goroutine都没有进入Wait状态。


原因是main goroutine执行的更快，并且在goroutine1/goroutine2添加锁并完成修改sharedRsc和发出广播信号之前已经获取了锁。

当子goroutine在调用Wait之前检查条件时，条件已经满足，因此无需再次调用Wait。


如果我们不在子goroutine中做检查呢？

```go
package main

import (
	"fmt"
	"sync"
)

var sharedRsc = false

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	m := sync.Mutex{}
	c := sync.NewCond(&m)
	go func() {
		// this go routine wait for changes to the sharedRsc
		c.L.Lock()
		for sharedRsc == false {
			fmt.Println("goroutine1 wait")
			c.Wait()
		}
		fmt.Println("goroutine1", sharedRsc)
		c.L.Unlock()
		wg.Done()
	}()

	go func() {
		// this go routine wait for changes to the sharedRsc
		c.L.Lock()
		fmt.Println("goroutine2 wait")
		c.Wait()
		fmt.Println("goroutine2", sharedRsc)
		c.L.Unlock()
		wg.Done()
	}()

	// this one writes changes to sharedRsc
	c.L.Lock()
	fmt.Println("main goroutine ready")
	sharedRsc = true
	c.Broadcast()
	fmt.Println("main goroutine broadcast")
	c.L.Unlock()
	wg.Wait()
}
```

我们会得到1个死锁。

```bash
main goroutine ready
main goroutine broadcast
goroutine2 wait
goroutine1 true
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0x414028, 0x19)
    /usr/local/go/src/runtime/sema.go:56 +0x40
sync.(*WaitGroup).Wait(0x414020, 0x40c108)
    /usr/local/go/src/sync/waitgroup.go:130 +0x60
main.main()
    /tmp/sandbox947808816/prog.go:44 +0x2c0

goroutine 6 [sync.Cond.Wait]:
runtime.goparkunlock(...)
    /usr/local/go/src/runtime/proc.go:307
sync.runtime_notifyListWait(0x43e268, 0x0)
    /usr/local/go/src/runtime/sema.go:510 +0x120
sync.(*Cond).Wait(0x43e260, 0x40c108)
    /usr/local/go/src/sync/cond.go:56 +0xe0
main.main.func2(0x43e260, 0x414020)
    /tmp/sandbox947808816/prog.go:31 +0xc0
created by main.main
    /tmp/sandbox947808816/prog.go:27 +0x140
```


为什么呢？

主goroutine（goroutine 1）先执行，停留在wg.Wait()，等待子goroutine的wg.Done()；而子goroutine（goroutine 6）直接调用cond.Wait而不判断条件。

当主goroutine调用Broadcast时，子goroutine还在cond.Wait中，而主goroutine在wg.Wait中阻塞，所以子goroutine永远不会被解救。


Wait将释放锁并等待另一个goroutine调用Broadcast或Signal来通知它恢复执行，但没有其他方法可以恢复。但是主goroutine已经调用了Broadcast并进入了等待状态，因此没有goroutine会解救仍在cond中的子goroutine。死锁。


因此，务必注意Broadcast必须在所有Wait之后运行（当然，可以通过条件判断是否进入Wait）。


我们来看看k8s中使用Cond实现的[FIFO](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/tools/cache/fifo.go)，它是如何处理条件的消费的。

```go
func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error) {
    f.lock.Lock()
    defer f.lock.Unlock()
    for {
        for len(f.queue) == 0 {
            // When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
            // When Close() is called, the f.closed is set and the condition is broadcasted.
            // Which causes this loop to continue and return from the Pop().
            if f.IsClosed() {
                return nil, FIFOClosedError
            }

            f.cond.Wait()
        }
        id := f.queue[0]
    f.queue = f.queue[1:]
    ...
    }
}

func NewFIFO(keyFunc KeyFunc) *FIFO {
    f := &FIFO{
        items:   map[string]interface{}{},
        queue:   []string{},
        keyFunc: keyFunc,
    }
    f.cond.L = &f.lock
    return f
}
```


Cond共享FIFO的锁，在Pop中，它首先添加锁`f.lock.Lock()`，并在`f.cond.Wait()`之前检查`len(f.queue)`是否为0，以防止2种情况。


1.  像来例子3一样，条件已经满足，无需等待
2.  条件已经满足，但是其他goroutine已经先行并在`f.lock`的锁定中阻塞；当获取锁并且锁定成功时，`f.queue`已被消耗为空，并且直接访问`f.queue[0]`将被访问越界。