> 本文是[LeakProf: Featherlight In-Production Goroutine Leak Detection](https://www.uber.com/en-SG/blog/leakprof-featherlight-in-production-goroutine-leak-detection/)的中文翻译版本，内容有删减

[Go](https://go.dev/) 是一种在微服务开发中广受欢迎的编程语言，其主要特点之一是对并发性的一流支持。鉴于其不断增长的受欢迎程度，Uber采用了该语言：[Go monorepo](https://www.uber.com/en-SE/blog/go-monorepo-bazel/) 是其开发平台的核心，其中包含了Uber的重要业务逻辑、支持库或基础设施的关键组件的大部分代码库。

Go的并发模型建立在轻量级线程——[_goroutines_](https://go.dev/doc/effective_go#goroutines)上。任何以"go"关键字为前缀的函数调用都会异步启动该函数。由于在Go代码库中使用goroutines的语法开销和资源需求较低，因此它们被广泛使用，程序通常可以同时涉及数十个、数百个或数千个goroutine。

两个或多个goroutine可以通过在通道上进行消息传递来相互通信，这是受到Hoare的[Communicating Sequential Processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes)的启发而形成的一种编程范式。虽然传统的共享内存通信仍然是一种选择，但Go开发团队鼓励用户更倾向于使用[_channels_](https://go.dev/doc/effective_go#channels)，并主张在使用时可以更好地避免数据竞争。你可以阅读[encourages](https://go.dev/blog/codelab-share)以了解更多信息。此外，Uber还发布了一篇关于[Go中数据竞争模式](https://www.uber.com/blog/data-race-patterns-in-go/)的博文。


Goroutine 泄露
---------------

[goroutine泄漏](https://betterprogramming.pub/common-goroutine-leaks-that-you-should-avoid-fe12d12d6ee#:~:text=When%20a%20new%20Goroutine%20is,background%20for%20the%20application's%20lifetime.)是goroutines的一个副作用。通道语义的一个关键组成部分是"阻塞"，即通道操作会使得goroutine的执行暂停，直到达成目标（即找到通信伙伴）。更具体地说，对于无缓冲通道，发送方在接收方到达通道之前一直阻塞，反之亦然。一个goroutine可能永远被阻塞在尝试发送或接收通道的过程中，这种情况被称为"goroutine泄漏"。当有过多的goroutine泄漏时，后果可能是[严重的](https://medium.com/golangspec/goroutine-leak-400063aef468)。泄漏的goroutine会消耗资源，例如未被释放或回收的内存。请注意，一旦缓冲区已满，有缓冲通道也可能导致goroutine泄漏。

程序错误（例如，复杂的控制流、早期返回、超时等）可能会导致goroutine之间的通信不匹配，其中一个或多个goroutine可能会被阻塞，此时不能通过创建其他goroutine会来解除阻塞。Goroutine泄漏会阻止垃圾回收器回收相关的通道、goroutine堆栈和永久阻塞的goroutine的所有可访问对象。在长时间运行的服务中，随着时间的推移，小的泄漏问题会加剧这个问题。

Go的发行版本在编译或运行时都没有提供直接的解决方案来检测goroutine泄漏。检测goroutine泄漏是非常复杂的，因为它们可能依赖于多个goroutine之间的复杂交互/交错，或者其他罕见的运行时条件。一些提出的静态分析技术[[1](https://github.com/nicolasdilley/Gomela), [2](https://github.com/cs-au-dk/goat), [3](https://github.com/system-pclub/GCatch)]容易出现不精确的情况，可能出现误报的问题。

其他提案，例如[goleak](https://github.com/uber-go/goleak)，在测试过程中采用动态分析，可能会揭示出多个阻塞错误，但其有效性取决于代码路径和线程调度的全面覆盖。在大规模的项目中进行详尽的覆盖是不可行的；例如，某些在生产环境中更改代码路径的配置可能并未被单元测试覆盖到。

因此，目前对于检测goroutine泄漏没有一种完美的解决方案。开发人员需要综合考虑静态和动态分析方法，并努力在测试过程中覆盖尽可能多的代码路径，以最大程度地减少goroutine泄漏的风险。

检测goroutine泄漏是复杂的，特别是在使用大量库、在运行时涉及数千个goroutine并使用大量通道的复杂生产代码中。迫切需要在生产代码中检测这些泄漏。这样的工具需要满足以下要求：

1. 它不应引入高额的开销，因为它将在生产负载中使用。高开销会影响服务级别协议（SLAs）并消耗更多的计算资源。

2. 它应具有较低的误报率；虚假的泄漏报告会浪费开发人员的时间。

一个实用、轻量级的解决方案：生产环境中的Goroutine泄漏
---------------

我们采用了一种实用的方法来检测生产环境中长时间运行的程序中的Goroutine泄漏，满足上述的准则。我们的前提和关键观察如下：

1. 如果一个程序存在大量的Goroutine泄漏，它最终会通过大量被阻塞在某些通道操作上的Goroutine数量变得可见。

2. 只有很少的源代码位置（涉及通道操作）导致了大部分的Goroutine泄漏。

3. 虽然不理想，罕见的Goroutine泄漏产生的开销较低，可以忽略不计。

Point #1 is justified by a spike in the number of goroutines for programs with leaks. Point #2 simply states that not all channel operations are leaky, but the source location of significant leak causes must be exercised frequently to expose the leak. Since any leaked goroutine persists for the remainder of a service’s lifetime, repeatedly encountering leaky scenarios eventually contributes to a large build-up of blocked goroutines. This is especially true for leaks caused by concurrency operations reachable via many different execution paths and/or in loops. Finally, point #3 is a practical consideration. If a leak-inducing operation is rarely encountered, the impact it has on memory build-up is less likely to be severe. Based on these pragmatic observations, we designed _LeakProf_, a reliable leak indicator with few false positives and a minimum runtime overhead.

Implementation of LeakProf
--------------------------

![][img-0]Figure 1: LeakProf architecture

LeakProf periodically profiles the call stacks of currently running goroutines (obtained with [pprof](https://github.com/google/pprof)). Inspecting the call stacks for particular configurations indicates whether a goroutine is blocked on specific operations such as channel send, channel receive, and select. These are relatively straightforward to recognize as well-known blocking functions in the Go runtime. After aggregating blocked goroutines by the source locations of channel operations, an egregious number of blocked goroutines at a single source location, determined by a configurable threshold, is treated as a potential goroutine leak.

Our approach is neither sound nor complete. It, respectively, incurs both false negatives (i.e., not all leaks are guaranteed to be found), and false positives (i.e., reports may not represent a real leak). False negatives are caused whenever a program does not exercise leaky scenarios at runtime, or if the number of leaks does not exceed the configurable threshold. Conversely, false positives lead to spurious reports whenever a high number of goroutines are deliberately blocked as a natural consequence of the intended program semantics, and not due to a leak (e.g., heartbeat operations with high latency). To improve filtering of potential false positives, we are continuously developing complementary lightweight heuristics based on static analysis. For example, reporting a suspicious **select** statement involves analyzing its AST to determine whether one of the **case** branches involves waiting on known non-blocking operations (e.g., involving tickers or timers provided by the Go standard library); if this property is satisfied, the supposed leak will not be reported, regardless of the number of blocked goroutines, as it is a definite false positive. It may also feature a configurable list of known false positives. Regardless, the approach works very well in practice to detect non-trivial leaks with a large impact on production services, as evidenced below.

When deployed at Uber, LeakProf leverages profiling information to collect blocked goroutine information and automatically notifies service owners for suspicious concurrency operations. Operations are deemed suspicious if the number of blocked goroutines exceeds a given threshold, and only if they are part of the Uber code base. The efficacy of this approach was proven quickly, finding 10 critical leaking goroutines, and only 1 false positive. Addressing 2 of the defects each resulted in a 2.5x and a 5x reduction in the service’s peak memory, with the service owners voluntarily reducing the container memory requirements by 25%.

![][img-1]Figure 2: Memory footprint example

Leaky Go Program Patterns
-------------------------

An analysis of goroutine leaks found in production has relieved the following common patterns of leaky code patterns.

### Premature Function Return

This leak pattern is caused whenever several goroutines are expected to communicate, but some code paths return prematurely without participating in the channel communication, leaving the other participant waiting forever. This occurs when communication partners do not account for all of each other’s possible execution paths.

##### Example

![][img-2]

The channel **c** (line 2) is used by the child goroutine (line 3) to send an error (lines 5 or 8). The corresponding receive operation on the parent thread (line 18) is preceded by several **if** statements which may **return** (lines 13 and 15) prior to waiting to receive from the channel on line 18. If the parent goroutine follows any of these returns, the child goroutine will block forever, irrespective of which send operation it performs. 

A possible solution to prevent the goroutine leak where a child goroutine sends a message on channel and the parent may not want to ignore it is to create the channel with a buffer size of 1. This allows the child sender to unblock on the communication operation, irrespective of the behavior of the parent receiver goroutine.

### The Timeout Leak

While this bug may be framed as a special case of the premature function return pattern, it deserves its own entry due to sheer ubiquity. This leak often appears when combining unbuffered channel usage with either [timers](https://pkg.go.dev/time) or [contexts](https://pkg.go.dev/context), and **select** statements. The timer or context is often used to short-circuit the execution of the parent goroutine and abort early. However, if the child goroutine is not designed to consider this scenario, it may lead to a leak.

##### Example

![][img-3]

Channel **done** (line 3) is used with a child goroutine (line 4). When the child goroutine sends a message (line 6), it blocks until another goroutine (ostensibly the parent) reads from **done**. Meanwhile, the parent waits at the **select** statement on line 8, until it synchronizes with the child (line 9), or when **ctx** times out (line 11). In the event of a context timeout, the parent goroutines returns through the case on line 11; as a result, there is no waiting receiver when the child sends. Consequently, the child will leak, as no other goroutine will ever receive from **done**.

This leak can similarly be averted by increasing the capacity of **done** to 1.

### The NCast Leak

This kind of leak occurs whenever the concurrent system is framed as a communication between many senders and a single receiver on the same channel. Moreover, if the receiver performs only one receive on the channel, all but one of the senders will block forever on the channel.

##### Example

![][img-4]

The channel **dataChan** (line 2) is passed as an argument to the goroutines created in the **for**-loop on line 4. Each of the child goroutines attempts to send a result to **dataChan**, but the parent only receives from **dataChan** once, and then exits the scope of its current function, at which point it loses the reference to **dataChan**. Since **dataChan** is unbuffered, any child which did not synchronize with the parent will block forever.

The solution is to increase the buffer of **dataChan** to the length of **items**. This preserves the property that only the first result sent to **dataChan** will be received by the parent thread, while allowing remaining child goroutines to unblock and terminate.

A more general form of this problem occurs when there are N senders and M receivers, N > M, and each receiver only performs one receive operation.

### Channel Iteration Misuse

This leaky pattern may occur when using the **range** construct with channels. Understanding this leak requires some familiarity with the [close operation](https://gobyexample.com/closing-channels) and how [range works with channels](https://gobyexample.com/range-over-channels). The leak is caused whenever a channel is iterated over, but the channel is never closed. This causes the **for** loop iterating over the channel to block indefinitely because the loop does not terminate unless the channel is closed; the **for** loop blocks once all items are received from the channel.

##### Example

![][img-5]

For brevity, we will borrow the producer-consumer dichotomy introduced in Communication contention. A channel, **queueJobs**, is allocated at line 3. The producers are the goroutines spawned in the **for** loop (line 3), where each sends a message (line 5). The  consumer (line 8) reads the messages by ranging over **queueJobs**. As long as an unconsumed message exists, the loop at line 9 will perform an iteration of the loop. The expectation is that once the producers no longer send messages, the consumer will exit the loop and terminate. However, in the absence of a close operation on the channel, **range** will simply block once no more messages are sent which leads to a leak.

Since the parent of the producers and consumers waits until all messages are delivered (via the [WaitGroup](https://gobyexample.com/waitgroups) construct), the solution is to add **close(queueJobs)** after **wg.Wait(),** or as a deferred statement immediately after **queueJobs** is defined. Once all messages are sent, the parent thread closes **queueJobs**, signaling to the consumer to stop iterating over **queueJobs**, and therefore terminate and be garbage collected.

Conclusions and Future Work
---------------------------

Goroutine leaks are a pervasive problem that are difficult to detect statically and can lead to a large waste in terms of computational resources. _LeakProf_ identifies those goroutine leaks that are egregious or accumulate over time in long-running production services. It detects goroutines blocked on channel operations via profiling programs followed by aggregating the goroutines blocked at the same source location, which is often a symptom of leak. When the number of leaking goroutines is large, LeakProf works well in raising an alarm.  

LeakProf’s efficacy was demonstrated quickly, given the number of leaks it found in a relatively short time frame. Another critical component is the insight gained by inspecting the code causing the reported leaks, which led to the discovery and synthesis of several problematic coding patterns. Notably, the discovered patterns showcase potential opportunities for the development of linters tailored for reporting and preventing leaks that would be caused by code conforming to the outlined patterns. Other future work includes devising better heuristics for determining the blocking goroutine threshold on a per-profile basis, with the intent to more effectively detect leaks in smaller programs, and further enhancements to the supplementary static analysis suite performed post-detection.

