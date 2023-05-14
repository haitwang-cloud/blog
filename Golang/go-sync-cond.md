> 本文是[How to properly use the conditional variable sync.Cond in Golang](https://www.sobyte.net/post/2022-07/go-sync-cond/)的中文翻译版本，内容有删减


Cond in Golang’s sync package implements a conditional variable that can be used in scenarios where multiple Readers are waiting for a shared resource ready (if there is only one read and one write, a lock or channel takes care of it).

Cond pooling point: multiple goroutines waiting, 1 goroutine notification event occurs.

Each Cond is associated with a Lock (`*sync.Mutex or *sync.RWMutex`), which must be added when modifying conditions or calling Wait methods, **protecting the condition**.

```
type Cond struct {
        // L is held while observing or changing the condition
        L Locker
        // contains filtered or unexported fields
}
```

```
func NewCond(l Locker) *Cond
```

Create a new Cond conditional variable.

```
func (c *Cond) Broadcast()
```

Broadcast will wake up **all** goroutines waiting for c.

Broadcast can be called with or without locking.

```
func (c *Cond) Signal()
```

Signal wakes up only **1** goroutine waiting for c.

Signal can be called with or without locking.

`Wait()` automatically releases `c.L` and hangs the caller’s goroutine. execution resumes later, and `Wait()` puts a lock on `c.L` when it returns.

`Wait()` does not return unless it is woken up by Signal or Broadcast.

Since `C.L` is not locked when `Wait()` first resumes, the caller usually does not assume that the condition is true when Wait returns.

Instead, the caller should call Wait in a loop. (Simply put, whenever you want to use a condition, you must add a lock.)

```
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```

The following example is a better illustration of how Cond is used.

[Playground1](https://play.golang.org/p/g2hyc2yDdJu)

```
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

The execution results are as follows.

```
goroutine1 wait
goroutine2 wait
main goroutine ready
main goroutine broadcast
goroutine2 true
goroutine1 true
```

goroutine1 and goroutine2 enter Wait state, after main goroutine in 2s after the resource is satisfied, after issuing broadcast signal, resume from Wait and determine whether the condition has indeed been satisfied (sharedRsc is not empty), satisfied then consume the condition and unlock, `wg.Done()` .

We make a modification to remove the 2s delay in the main goroutine.

The code will not be posted.

[Playgroud2](https://play.golang.org/p/9svW4WI4lXK)

The execution result is as follows.

```
main goroutine ready
main goroutine broadcast
goroutine2 true
goroutine1 true
```

It is interesting to note that neither goroutine enters the Wait state.

The reason is that the main goroutine executes faster and has already acquired the lock before goroutine1/goroutine2 adds the lock and finishes modifying sharedRsc and signaling Broadcast.

When the child goroutine checks the condition before calling Wait, the condition is already satisfied, so there is no need to call Wait again.

What if we don’t do checksum in the subgoroutine?

[Playground3](https://play.golang.org/p/vtHEVQCm6NC)

We would get 1 deadlock.

```
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

Why?

The main goroutine (goroutine 1) executes first and stays in wg.Wait(), waiting for wg.Done() of the child goroutine; while the child goroutine (goroutine 6) calls cond.Wait directly without judging the condition.

Wait will release the lock and wait for the other goroutine to call Broadcast or Signal to notify it to resume execution, but there is no other way to resume. But the main goroutine has already called Broadcast and entered the wait state, so no goroutine will rescue the child goroutine that is still in cond. Deadlock.

Therefore, be sure to note that Broadcast must come after all the Wait (of course, it is possible to decide whether to go into Wait by conditional judgment).

Let’s take a look at the [FIFO](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/tools/cache/fifo.go) implemented in k8s using Cond, which How to handle the consumption of conditions.

```
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

Cond shares the FIFO’s lock, in Pop, it will add lock `f.lock.Lock()` first, and before `f.cond.Wait()`, it will check if `len(f.queue)` is 0 to prevent 2 cases.

1.  as in example 3 above, the condition is satisfied, no need to wait
2.  The condition is satisfied when waking up, but other goroutines have gotten there first and blocked in the locking of `f.lock`; when the lock is obtained and the locking is successful, `f.queue` has been consumed as empty, and direct access to `f.queue[0]` will be accessed out of bounds.