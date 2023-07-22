> 本文是[LeakProf: Featherlight In-Production Goroutine Leak Detection](https://www.uber.com/en-SG/blog/leakprof-featherlight-in-production-goroutine-leak-detection/)的中文翻译版本，内容有删减

[Go](https://go.dev/) 是一种在微服务开发中广受欢迎的编程语言，其主要特点之一是对并发性的一流支持。鉴于其不断增长的受欢迎程度，Uber采用了该语言：[Go monorepo](https://www.uber.com/en-SE/blog/go-monorepo-bazel/) 作为开发平台的核心，其中包含了Uber的重要业务逻辑、支持库或基础设施的关键组件的大部分代码库。

Go的并发模型建立在轻量级线程——[_goroutines_](https://go.dev/doc/effective_go#goroutines)上。任何以"go"关键字为前缀的函数调用都会异步启动该函数。由于在Go代码库中使用goroutines的语法开销和资源需求较低，因此它们被广泛使用，程序通常可以同时涉及数十个、数百个或数千个goroutine。

两个或多个goroutine可以通过在通道上进行消息传递来相互通信，这是受到Hoare的[Communicating Sequential Processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes)的启发而形成的一种编程范式。虽然传统的共享内存通信仍然是一种选择，但Go开发团队鼓励用户更倾向于使用[_channels_](https://go.dev/doc/effective_go#channels)，并主张在使用时可以更好地避免数据竞争。你可以通过[encourages](https://go.dev/blog/codelab-share)了解更多信息，此外Uber还发布了一篇关于[Go中数据竞争模式](https://www.uber.com/blog/data-race-patterns-in-go/)的博文。


Goroutine 泄露
---------------

[goroutine泄漏](https://betterprogramming.pub/common-goroutine-leaks-that-you-should-avoid-fe12d12d6ee#:~:text=When%20a%20new%20Goroutine%20is,background%20for%20the%20application's%20lifetime.)是goroutines高并发的一个副作用。[_channels_](https://go.dev/doc/effective_go#channels)语义的一个关键组成部分是"阻塞"，即[_channels_](https://go.dev/doc/effective_go#channels)操作会使得goroutine的执行暂停，直到达成目标（即找到通信伙伴）。更具体地说，对于无缓冲[_channels_](https://go.dev/doc/effective_go#channels)，发送方在接收方到达通道之前一直阻塞，反之亦然。一个goroutine可能永远被阻塞在尝试发送或接收通道的过程中，这种情况被称为"goroutine泄漏"。当有过多的goroutine泄漏时，后果可能是[严重的](https://medium.com/golangspec/goroutine-leak-400063aef468)。泄漏的goroutine会消耗资源，例如未被释放或回收的内存。请注意，一旦缓冲区已满，有缓冲[_channels_](https://go.dev/doc/effective_go#channels)也可能导致goroutine泄漏。

程序错误（例如，复杂的控制流、早期返回、超时等）可能会导致goroutine之间的通信不匹配，其中一个或多个goroutine可能会被阻塞，此时不能通过创建其他goroutine会来解除阻塞。Goroutine泄漏会阻止垃圾回收器回收相关的[_channels_](https://go.dev/doc/effective_go#channels)、goroutine堆栈和永久阻塞的goroutine的所有可访问对象。在长时间运行的服务中，随着时间的推移，小的泄漏会加剧这个问题。

Go的发行版本在编译或运行时都没有提供直接的解决方案来检测goroutine泄漏。如何检测goroutine泄漏是非常复杂的，因为它们可能依赖于多个goroutine之间的复杂交互，并且只在某些罕见的运行时会触发。一些提出的静态分析技术[[1](https://github.com/nicolasdilley/Gomela), [2](https://github.com/cs-au-dk/goat), [3](https://github.com/system-pclub/GCatch)]容易出现不精确的情况，可能出现误报的问题。

其他提案，例如[goleak](https://github.com/uber-go/goleak)，主要思路是在测试过程中采用动态分析，然后揭示出多个阻塞错误，但其有效性取决于对代码路径和线程调度的单元测试覆盖。在大规模的项目中进行详尽的单元测试覆盖是不可行的；例如，某些在生产环境中更改代码路径的配置可能并未被单元测试覆盖到。

因此，目前对于检测goroutine泄漏没有一种完美的解决方案。开发人员需要综合考虑静态和动态分析方法，并努力在测试过程中覆盖尽可能多的代码路径，以最大程度地减少goroutine泄漏的风险。

检测goroutine泄漏是复杂的，特别是在使用大量库、在运行时涉及数千个goroutine并使用大量通道的复杂生产代码中。在生产代码中检测这些泄漏的工具需要满足以下要求：

1. 它不应引入高额的开销，因为它将在生产负载中使用。高开销会影响**SLAs**并消耗更多的计算资源。

2. 它应具有较低的误报率；虚假的泄漏报告会浪费开发人员的时间。

一个实用、轻量级的解决方案：生产环境中的Goroutine泄漏
---------------

我们采取实际方法来检测在生产环境中长时间运行的程序中的Goroutine泄漏，以满足前面提到的标准。我们的前提和关键观察如下：：

1. 如果一个程序存在大量的Goroutine泄漏，它最终会通过大量被阻塞在某些通道操作上的Goroutine数量变得可见。

2. 只有很少的源代码位置（涉及通道操作）导致了大部分的Goroutine泄漏。

3. 虽然不理想，罕见的Goroutine泄漏产生的开销较低，可以忽略不计。

第一点通过观察到有泄漏的程序中Goroutine数量的激增得到了证实。第二点简单地说明并非所有的通道操作都会导致泄漏，只有导致泄漏的代码被反复执行，才可能暴露这种泄漏。由于任何泄漏的Goroutine会持续存在于服务的声明周期中，反复遇到泄漏最终会导致大量积压的阻塞Goroutine。这对于由许多不同执行循环中的并发操作引起的泄漏尤其成立。最后，第三点是实际考虑因素。如果导致泄漏的操作很少遇到，它对内存积聚的影响可能不会很严重。基于这些实用的观察结果，我们设计了_LeakProf_，它是一个可靠的泄漏指示器，几乎不会产生误报，并且运行时开销最小化。

LeakProf的实现
--------------------------

![](./pics/leak-1.webp)Figure 1: LeakProf architecture


LeakProf定期对当前正在运行的goroutine进行调用栈分析（通过[pprof](https://github.com/google/pprof)获得）。检查特定配置的调用栈可以指示一个goroutine是否在特定操作（如通道发送、通道接收和select）上被阻塞。这些阻塞函数在Go运行时中是相对容易识别的。通过统计在按照通道操作的源位置汇总阻塞的goroutine，如果在单个位置上有大量的被阻塞goroutine，且超过了可配置的阈值，LeakProf会将其视为潜在的goroutine泄漏。

我们的方法既不完全准确，也不完备。分别存在着两类错误，即假阴性（即未必能找出所有的泄漏）和假阳性（即报告可能不代表真实的泄漏）。假阴性出现在程序在运行时没有出现泄漏场景，或者泄漏数量未超过可配置的阈值时。相反，假阳性会导致虚假的报告，当大量的goroutine由于程序意图的语义而被有意识地阻塞，而不是由于泄漏导致的时候（例如，具有高延迟的心跳操作）。为了改进对潜在假阳性的过滤，我们正在持续开发基于静态分析的轻量级启发式方法。例如，报告一个可疑的**select**语句涉及分析其AST以确定其中一个**case**分支是否涉及等待已知的非阻塞操作（例如，涉及Go标准库提供的定时器或计时器）；如果满足此条件，则不会报告该泄漏，无论有多少阻塞的goroutine，因为它肯定是假阳性。还可以配置已知假阳性的列表。不过，尽管如此，这种方法在实践中非常有效，可以检测到对生产服务产生重大影响的非平凡泄漏，如下所示。

在Uber部署时，LeakProf利用性能分析信息来收集被阻塞的goroutine信息，并自动通知服务所有者进行可疑并发操作的检查。只有当被阻塞的goroutine数量超过给定的阈值，并且它们是Uber代码库的一部分时，这些操作被视为可疑。这种方法的有效性得到了快速验证，它迅速发现了10个关键的泄漏goroutine，并且仅有1个假阳性。修复其中2个缺陷分别导致了服务峰值内存的2.5倍和5倍的减少，服务所有者自愿将容器内存需求减少了25%。

![](./pics/leak-2.webp)
Figure 2: Memory footprint example

泄漏代码模式
-------------------------

在生产环境中对Goroutine泄漏的分析揭示了以下常见的泄漏代码模式。
### 过早函数返回

这种泄漏模式在几个goroutine预期进行通信时发生，但是某些代码路径过早地返回而没有参与通道通信，导致另一方永远等待。这种情况发生在通信双方没有考虑彼此可能的所有执行路径时。

##### Example

![](./pics/leadprof-3.webp)


通道c（第2行）被子goroutine（第3行）用于发送错误消息（第5行或第8行）。父线程上的相应接收操作（第18行）之前有几个if语句，可能在第18行等待从通道接收之前就执行return操作（第13行和第15行）。如果父goroutine执行这些return语句中的任何一个，子goroutine将永远阻塞，无论它执行哪个发送操作。

防止goroutine泄漏的一种可能解决方案是在创建通道时使用缓冲大小为1。这允许子goroutine的发送者在通信操作上解除阻塞，而不受父接收goroutine行为的影响。

### 超时泄露

虽然这个bug可以被看作是过早函数返回模式的特例，但由于其普遍性，它值得独立列出。这种泄漏经常出现在将无缓冲通道与timers或contexts，以及select语句相结合的情况下。定时器或上下文通常用于短路(short-circuit)父goroutine的执行并提前终止。然而，如果子goroutine没有考虑到这种情况，可能会导致泄漏。

##### Example

![](./pics/leakprof-4.webp)

通道**done**（第3行）与子goroutine（第4行）一起使用。当子goroutine发送消息（第6行）时，它会阻塞，直到另一个goroutine（可能是父goroutine）从**done**通道读取。同时，父goroutine在第8行的**select**语句处等待，直到与子goroutine同步（第9行），或者当**ctx**超时（第11行）时。在上下文超时的情况下，父goroutine通过第11行的case返回；结果是，当子goroutine发送时，没有等待的接收者。因此，子goroutine会发生泄漏，因为没有其他goroutine会从**done**接收。

同样，通过将**done**的容量增加为1，可以避免这种泄漏。

### 广播泄露

这种类型的泄漏发生在并发系统被构建为在同一个通道上有**多个发送者**和**单个接收者**之间的通信时。此外，如果**单个接收者**在通道上只执行一次接收操作，除了一个发送者外，其他所有发送者都将永远在通道上阻塞

##### Example

![](./pics/leakprof-5.webp)

通道**dataChan**（第2行）被作为参数传递给第4行的**for**循环中创建的goroutine。每个子goroutine都试图向**dataChan**发送一个结果，但父goroutine只从**dataChan**接收一次，然后退出其当前函数的作用域，此时它失去了对**dataChan**的引用。由于**dataChan**是无缓冲的，任何未与父goroutine同步的子goroutine都将永远阻塞。

解决方案是将**dataChan**的缓冲区增加到**items**的长度。这保留了只有第一个结果发送给**dataChan**会被父线程接收的特性，同时允许其余的子goroutine解除阻塞并终止。

这个问题的更一般形式是当有N个发送者和M个接收者，其中N > M，并且每个接收者只执行一次接收操作时。

### 通道迭代错误使用

这种泄漏模式可能发生在使用range结构与通道时。理解这种泄漏需要对[关闭操作](https://gobyexample.com/closing-channels) 和[range与通道的工作原理]((https://gobyexample.com/range-over-channels))有一定的了解。泄漏是在对通道进行迭代时发生的，但是通道从未被关闭。这会导致for循环对通道的迭代无限期地阻塞，因为除非通道关闭，否则循环不会终止；for循环在从通道接收所有项目后会被阻塞。

##### Example

![](./pics/leakprof-6.webp)

为了简洁起见，我们将借用在"Communication contention"（通信竞争）中引入的生产者-消费者问题。在第3行分配了一个名为**queueJobs**的通道。生产者是在**for**循环（第3行）中生成的goroutine，在其中每个生产者发送一条消息（第5行）。消费者（第8行）通过遍历**queueJobs**来读取消息。只要存在未消费的消息，第9行的循环将执行一次迭代。预期的结果是，一旦生产者不再发送消息，消费者将退出循环并终止。然而，由于通道上没有执行close操作，**range**在没有更多消息发送时将阻塞，从而导致泄漏。

由于生产者和消费者的父goroutine在所有消息被交付之前等待（通过[WaitGroup](https://gobyexample.com/waitgroups)构造），解决方法是在**wg.Wait()**之后添加**close(queueJobs)**，或者作为延迟语句。一旦所有消息被发送，父goroutine会关闭**queueJobs**，向消费者发出信号停止遍历**queueJobs**，从而终止并被垃圾回收。

结论和未来工作
---------------------------
Goroutine泄漏是一个普遍存在的问题，在静态检测方面很难发现，可能会导致计算资源的大量浪费。_LeakProf_能够识别那些在长时间运行的生产服务中严重或随着时间累积的Goroutine泄漏。它通过对程序进行性能分析来检测在通道操作上被阻塞的Goroutine，并通过汇总在相同源位置被阻塞的Goroutine来发现泄漏的迹象。当泄漏的Goroutine数量很大时，LeakProf能够有效地发出警报。

LeakProf的有效性在相对较短的时间内发现了许多泄漏问题，其中另一个关键组件是通过检查导致报告泄漏的代码而获得的洞察力，这导致了对几种问题编码模式的发现和综合。值得注意的是，发现的这些模式为开发定制用于报告和防止符合这些模式的代码导致的泄漏的linter提供了潜在的机会。其他未来的工作包括制定更好的启发式方法来确定每个性能分析中阻塞Goroutine的阈值，以更有效地检测较小程序中的泄漏，以及进一步完善在检测后进行的辅助静态分析套件。