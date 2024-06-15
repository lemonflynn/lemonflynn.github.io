---
layout: post
title:  "multi-core memory consistency and cache coherence"
date:   2024-6-15 7:00:16 +0800
categories: kernel 
---
**作者: flynn**

## 引言

最近积累了很多技术疑点，如下：

- 平时在翻阅 ARM 技术手册的时候，会看到 ARM 采用的是 weak memory model，而 x86 则采用 strong memory model，这是什么意思呢？

- 多个处理器在抢同一个 spin lock 时，是如何保证只有一个 CPU 得到并占用了这个 lock，具体细节上是如何实现呢？

- 多个处理器访问同一块内存的时候，一个 CPU 更新数据后，另一个 CPU 是如何看到第一个 CPU 写的数据的？
- 在 KVM x86代码中，有看到当虚拟机包含了 noncoherent dma 设备的时候，kvm 为 guest 设置的内存类型要从原本的 write-back 改为 uncached，这是为什么呢？

接下来，我们就来慢慢分析。

## 目录

1. memory consistency model
2. cache coherence 
3. snooping coherence protocol



## memory consistency model

memory consistency model 是什么？要回答这个问题，先看一个例子：

<img src="/assets/coherence/image-20240531125341715.png" alt="image-20240531125341715" style="zoom:43%;" />

由于现代处理器大多是乱序之行的，所以指令执行的顺序并不一定和程序指令的顺序保持一致，所以，S2 有可能先于 S1 被执行，如下图：

<img src="/assets/coherence/image-20240531125650057.png" alt="image-20240531125650057" style="zoom:43%;" />

这样以来，r2 的值就变成了 0，而不是我们预想的 NEW。

再看一个例子：

<img src="/assets/coherence/image-20240531130054548.png" alt="image-20240531130054548" style="zoom:33%;" />

r1 和 r2 的值可能是什么呢？

(0,  NEW)

(NEW, 0)

(NEW, NEW)

上面三张情况比较好理解，那有可能是 **(0, 0)** 吗？答案是，在 x86 架构下，是完全有可能的，由于在 x86 处理器中，每个 CPU 核一般都会有 write-buffer 来缓存来自 load-store unit 的数据，所以当 store 指令执行完时，并不代表数据已经更新到 cache 子系统中了，有可能位于 write-buffer 中，其内容对其他 CPU 核来说，不可见。

所以，为了消除这些模糊的行为，我们定义内存模型 (memory model) 来解决，有了内存模型之后，我们就可以做到：

1. 程序员可以预期程序的执行顺序
2. CPU 架构师可以以此来优化 CPU 的设计

我们来看一个最符合直觉的内存模型，**SC(sequential consistency)** 内存模型

### SC 内存模型

SC 内存模型特性如下：

<img src="/assets/coherence/image-20240531131420070.png" alt="image-20240531131420070" style="zoom: 27%;" />

通过下图，可以看出所有的 **load**， **store**， **RMW** 指令的访存顺序都严格遵守其指令顺序

<img src="/assets/coherence/image-20240531131850071.png" alt="image-20240531131850071" style="zoom:20%;" />

那么如何来实现这样的一个模型呢？

<img src="/assets/coherence/image-20240531132431494.png" alt="image-20240531132431494" style="zoom:33%;" />

最简单的方式是在 memory 子系统和各个 CPU 核之间加一个 switch，用来保证，同时只有一个 CPU 可以访问内存子系统，并且要求 CPU 核不能乱序执行，可以看出，这样的实现方式是很严格的，必然对性能造成很大的影响。

如果把 switch 和 memory 子系统看成一个黑盒，则这个黑盒可以称之为 cache coherent memory system

<img src="/assets/coherence/image-20240531133759911.png" alt="image-20240531133759911" style="zoom:33%;" />

当然，cache coherent memory system 不可能这么笨，每次只服务于一个 CPU 核，我们把这个黑盒拆开一点 ，把属于每个 CPU 核的 L1 缓存暴露出来，来看看它有什么神奇之处：

<img src="/assets/coherence/image-20240601102201222.png" alt="image-20240601102201222" style="zoom:37%;" />

我们知道，根据设计需要，缓存可以划分成 1- N 组，每组由多个缓存行组成，每个缓存行( cache block) 由标志位、tag 和数据块组成，我们在每个 cache block 中增加如下两个状态位，就可以优化这个简单的设计：

1. Modified：只有一个 CPU 可以写这个 cache block
2. Shared：可以有一个或多个 CPU 读这个 cache block

看下面的例子：

<img src="/assets/coherence/image-20240601103013935.png" alt="image-20240601103013935" style="zoom:33%;" />

C1 和 C2 写不同的缓存行，而 C3 和 C4 读同一个缓存行，很明显 cache-coherent memory system 可以在同一个时刻去响应这些请求，因为他们之间并不会造成任何冲突。通过这个改进，cache-coherent memory system 显然要比之前提到的 switch 模型要高级，毕竟可以做到同时服务多个 CPU 核。

原子操作(RMW)通常被用在多线程的程序中，例如 atomic_inc、compare-and-swap，test-and-set 等，我们熟知的 spin-lock 就会使用原子操作来获取锁，1)读 lock 的值，2)判断 lock 是否被锁，3)如果 lock 没有被锁则更新 lock 的值，虽说是单独的三个步骤，但不允许其他处理器读取或修改 lock 的值，所以称之为原子操作。

对于 SC 内存模型来说，实现原子操作最简单的方式是锁住整个内存总线，这样无疑最简单，但是效率也最低。更高效的做法是：

1. cpu core 请求设置包含数据的 cache block 为 M-state
2. cpu core 进行 load 操作
3. cpu core 进行运算
4. cpu core 进行 store 操作
5. 如果有其他 cpu core 也对这个 cache block 发来 request，则阻塞响应，直到 4 的 store 结束

### TSO 内存模型

SC 内存模型虽然简单直接，但是性能比较低，例如一个 CPU 核执行 store 时，只有当 store 的值写入到内存子系统（包含 cache ）中时该 store 指令才会 retire，如果此时内存子系统并没有准备好响应 store 请求（对应的缓存行不是 M state 或者 缓存中就没有对应的数据），此时就需要 stall pipeline。

可是处理器其实并不需要等 store 指令真正的 retire 后，才能继续执行，于是，可以引入一个 FIFO 类型的 write-buffer，当 store 指令执行的时候，可以将 store 指令的地址和数据直接放入 write-buffer，CPU 去继续执行后序的指令，内存子系统会依次去处理 write-buffer 中的所有 store 指令，每处理一个 store 指令，则 retire 一个 store 指令，于是，write-buffer 起到了一个缓冲区的作用。

当每个 CPU 都有自己的 write-buffer 后，其内容对其他 CPU 不可见，这就会引入新的问题：

- 如果在同一个 CPU 上先执行 store 再执行 load 指令，load 指令会先去 write-buffer 中查找，是否有相同地址的 store 指令，如果有，则直接从其中拿数据， 这也就 ***bypass***。

- 假如 CPU1 先执行 `x = NEW` , 接着 CPU2 去读 x 的值，可能会读到旧的值，因为此时 CPU1 的 store 指令可能还位于 write-buffer 中，并没有更新到内存子系统，如下图所示：

<img src="/assets/coherence/image-20240601134701334.png" alt="image-20240601134701334" style="zoom:33%;" />

这种情况明显是违背 SC 内存模型下面这条规则：

- If S(a) <p L(b)  =>  S(a) <m L(b)  /\*Store -> Load*/

大多数情况下，我们还是想保持这样的规则，于是，引入 ***FENCE*** 指令，其含义是 FENCE 指令之前的指令必须在 retire 以后，才能执行 FENCE 指令之后的指令。FENCE 指令最简单的实现方式是，在执行 FENCE 指令时，等待 write-buffer 为空之后，确保前序 store 指令都完 retire 以后，才 retire FENCE 指令。

此时的内存模型，就是 **TSO**(Total store order) 内存模型，其 load，store，RMW，FENCE 指令的访存顺序如下图所示：

<img src="/assets/coherence/image-20240601141249078.png" alt="image-20240601141249078" style="zoom:33%;" />

### XC 内存模型

对于 TSO 内存模型，Load-Load，Load-Store，Store-Store 顺序的指令，其访存顺序依然严格遵守其指令顺序。其实很多时候也是不必要的 ，例如下图：

<img src="/assets/coherence/image-20240601141916544.png" alt="image-20240601141916544" style="zoom:33%;" />

S1和 S2，L2 和 L3 执行的先后顺序，并不影响程序的正确性，完全可以乱序执行。S1, S2 与 S3 之间可以通过 FENCE 指令来保证正确性，其 load，store，RMW，FENCE 指令的访存顺序如下图所示：

![image-20240601142207125](/assets/coherence/image-20240601142207125.png)

- **A** 表示如果是相同的地址，则访问内存的顺序遵循指令顺序 ，否则乱序执行
- **B** 表示如果是相同的地址，在通过 **bypass** 或者值，否则乱序执行

## cache coherence protocol

memory consistency model (内存一致性模型)厘清了一个 **CPU 核心**内部，不同访存指令被执行的顺序，以及其执行结果何时适当的被其他 CPU 核观察到。为了实现这个模型，我们需要一个 Cache-Coherent memory system (缓存一致性系统) 来处理多个处理器同时读写同一内存地址的情形。如下面这个常见的例子：

<img src="/assets/coherence/image-20240610161143183.png" alt="image-20240610161143183" style="zoom:30%;" />

在第 3 时刻，处理器A 更改了 X 的值为 0，不过处理器B 的缓存中还保留着旧的数据 1，那么，如果处理器B 接着去读 X 的值时，就会拿到旧的数据，这就会引起程序执行出错。这类情形非常常见，比如一个 CPU1 在等待一个全局变量被置 1，CPU2 在某个时刻将全局变量置1，可是由于 CPU1 一直读取的是其缓存中的旧数据，导致它就会一直等待。

那么，如何确保一个缓存系统是一致性的呢？，对于同一块内存，只需要满足如下两点：

1. 同一时刻，只有一个处理器可以写内存，或者可以有一个或多个处理器读内存
2. 处理器读到的数据，必须是被最新写到内存的数据，时刻3，处理器应该读到被 Core1 在时刻2 写的数据，而不是被 Core3 在时刻1 写的数据

<img src="/assets/coherence/image-20240610163210973.png" alt="image-20240610163210973" style="zoom:33%;" />

### 系统组成

当代的多核处理器中，每个 CPU 核都有自己独有的 L1/L2 cache，L3 cache 则被所有处理器共用，L3 cache 也被称为  last level cache (LLC)，为了方便解释，我们假设每个 CPU 核都只有一级缓存，那么缓存一致性系统的结构，如下图所示：

<img src="/assets/coherence/image-20240610163928843.png" alt="image-20240610163928843" style="zoom:37%;" />

其中，红框圈中的多个 Cache controller 和 Memory controller 组成了一个分布式系统，他们需要各司其职，同时进行不一样的工作，来确保实现缓存一致性。

Cache controller 的事情想对要多一些，它不仅需要服务来自处理器的 Load/Store 等指令，还需要响应来自内部互联网络上，其他控制器的请求，如下图：

<img src="/assets/coherence/image-20240610164710345.png" alt="image-20240610164710345" style="zoom:43%;" />

具体来看，来自处理器的指令有下面这些：

![image-20240610164818543](/assets/coherence/image-20240610164818543.png)

来自内部互联网络上的请求如下：

![image-20240610164903410](/assets/coherence/image-20240610164903410.png)

而内存控制器的工作，则简单的多，它只需要被动的响应来自内部互联网络上的请求就可以了。

### 分析方法

在 SC 内存模型一节中，我们为每个缓存行，引入了状态位，如此一来，cache coherent memory system 就可以变得 ***聪明*** 起来，就有可能同时响应多个处理器的请求，至于它可以变得有多 ***聪明***，就和状态位的设计有关，我们先来看个简单的例子，假设缓存行只有 **Invalid** 和 **Valid** 两个状态，共有三种类型的总线事务

1. Get: 请求获得某个物理地址对应的缓存块数据
2. DataResp: 传输缓存块数据
3. Put: 请求将某个物理地址对应的缓存块数据更新到 LLC 中

每个缓存行中的状态机如下图，Own-Put/Get 表示接收到来自自身控制器的请求，Ohter-Put/Get 表示接收到来自其他控制器的请求：

<img src="/assets/coherence/image-20240610172532630.png" alt="image-20240610172532630" style="zoom:33%;" />

我们使用表格来展示各个状态之间的转换细节，缓存控制器中的状态转换如下图：

<img src="/assets/coherence/image-20240610172901006.png" alt="image-20240610172901006" style="zoom: 33%;" />

表格有三行，每行代表一个状态，其中 IVD 是中间状态，表示由状态 I 转为 V，但是此时还没有收到数据。有多列，每一列表示一种事件类型，表格中的内容表示，处于某个状态时，接收到某个事件时，该做什么响应，例如`Send DataResp/I`  表示当处于 Valid 状态时，如果接收到来自其他处理器的 Get 请求，就需要发送 DataResp 并跳到状态 I，绿色格表示不可能出现的情况，空白格表示不做任何响应。

内存控制器中的状态转化比较简单，如下图：

<img src="/assets/coherence/image-20240610173122395.png" alt="image-20240610173122395" style="zoom:33%;" />

## snooping coherence protocol

上一节，我们用仅有两个状态的 cache coherence protocol，介绍了如何设计和分析协议的工作流程，现在，我们来研究一下稍微复杂一些的 coherence protocol。

假设我们的内部互联网络是总线结构，所有的 coherence 控制器同时监听内部互联网络上的消息，且消息到达各个控制器的顺序都是一致的，各个控制器有选择的去响应消息，同时维护自身内部的状态机。为了方便分析，我们先为总线添加如下两个限制：

1. **Atomic request**: 由于总线仲裁的原因，request 可能得不到总线使用权，此时 stall cpu pipeline
2. **Atomic transaction**: 在 request 还没有完成以前(DataResp)，不能接受新的request

一个总线事务，一般包括 request，response，data(可有可无)，其大致工作流程如下图：

![snoop_bus](/assets/coherence/snoop_bus.gif)

缓存行都有如下属性，不同的协议会使用不同的组合来优化设计：

- validity: 有来自 LLC 的最新数据，可读，只有当同时是 exclusive 时可写
- Dirtiness: 被 cpu 修改了，不同于 LLC
- Exclusive: 数据只存在于此缓存中
- Ownership: cache block 所有者，需要响应其他 request

### MSI protocol

最简单的 **MSI** 协议，其 cache controller 中，有如下三种状态：

**M**: modified (Valid/Exclusive/Owner, possibly Dirty)

- 数据只在此 cache 中，当被 evict 时，需要写回数据到 LLC

- cache block 是有效的，有可能是脏的，可读可写
- 需要对这个 block 的 GetS/GetM 进行响应

**S**: shared(Valid)

- 数据位于一个或多个 cache 中

- cache block 是有效的，但是只读

**I**: invalid

其对应的状态机转化如下图：

<img src="/assets/coherence/image-20240613084559677.png" alt="image-20240613084559677" style="zoom:33%;" />

Memory controller 中也有三种状态

**I**: 所有位于 cache 中的 block 都是 invalid 状态

- 需要对这个 block 的 GetS/GetM 进行响应

**S**: block 位于一个或多个 cache 中，且都是 shared 状态

- 需要对这个 block 的 GetS/GetM 进行响应

**M**: block 位于一个 cache 中，且是 modify 状态

- 不需要对这个 block 的 GetS/GetM 进行响应

状态机如下图：

<img src="/assets/coherence/image-20240613084809717.png" alt="image-20240613084809717" style="zoom:33%;" />

cache controller 中的状态转换细节如下图：

<img src="/assets/coherence/image-20240614075712368.png" alt="image-20240614075712368" style="zoom:33%;" />

memory controller 的转换细节如下：

<img src="/assets/coherence/image-20240614075747295.png" alt="image-20240614075747295" style="zoom:33%;" />

下面我们来看一个例子，当两个 cpu 同时访问同一个 cache block 的情形：

<img src="/assets/coherence/image-20240614080001526.png" alt="image-20240614080001526" style="zoom:33%;" />

实际上，之前提到的总线上 **Atomic request** 这一限定在实际中并不会得到满足，因为如果当缓存控制器一直得不到总线使用权的时候，就需要一直阻塞，进而阻塞了 CPU 的 pipeline 影响后续程序的执行，所以，需要在高速缓存控制器和低速总线（因为总线是被多个控制器共用的）之间加入一个 queue，这样一来，缓存控制器可以把总线请求放到 queue 中之后，直接返回，就不会阻塞 CPU 的 pipeline 了。这也引出新的问题，当 request 还在 queue 中的时候，如何响应 Own-request 和 Other-request ?

![atomic_request](/assets/coherence/atomic_request.gif)

所以需要引入新的中间状态，例如，ISAD , 表示要从 I 切换到 S，但是此时请求还在 queue 中，没有通过总线广播到其他其他控制器，那么，如何判断请求已经离开了 queue 呢？答案是当缓存控制器从总线上接收到自己的请求时，此时，状态就可以由 ISAD 切换到 ISD。MIA 表示缓存控制器要将状态从 M 切换到 I，但是该请求还处于 queue 中，于是该缓存行的状态其实还是 M，需要响应来自其他处理器的请求，IIA 的状态含义稍微复杂一些，需要结合内存控制器的状态转换来理解，这里先暂且按下不表。

同样，**atomic transcation** 这一限定也会影响内部互联网络的使用率，真实情况是，在上一个 request 还没有完收到 response 的时候，总线上下一个 request 就已经来了，这也会带来更多的复杂度，这里，也暂时先不讨论。

### MSIE protocol

程序中“先读再改写”这种情况随处可见，例如，`if(a > 10) a = 0; `如果仅用 **MSI** 协议，则缓存行的状态会按 `I->S->M` 进行转换，随着每次的状态切换，都伴随着数据传输，这明显是不必要的，所以引入 **Exclusive** 状态，当 cache controller 收到来自处理器对地址 a 的 GetS 请求时，如果此时其他处理器的缓存中没有地址 a 的拷贝，则进入 **Exclusive** 状态，表明地址 a 的数据是独有的，对它进行写操作不会对其他处理器造成任何影响，所以当 cache controller 再收到来自处理器对地址 a 的 GetM 请求时，状态可以直接从 E 切换到 M，而不用像总线发送任何请求，从而减小了总线流量。如下图：

MSI 协议的做法：

![msie_59](/assets/coherence/msie_59.gif)

MSIE 协议的做法：

![msie_2](/assets/coherence/msie_2.gif)

MSIE 缓存控制器的状态机如下图：

<img src="/assets/coherence/image-20240614082909986.png" alt="image-20240614082909986" style="zoom:33%;" />

这里留一个问题, 缓存控制器是如何知道某个地址的数据没有处于其他处理器的缓存中的？

### MOSI protocol

对于同一块内存，一个 cpu 写，另一个 cpu 读这种情况也非常常见，对于 MSI 协议来说，假如 CPU1 执行了 `a = 1` 之后，CPU 2 执行了 `b = a`，则对于 CPU1 来说，包含 a 的缓存行状态需要由 **M -> S**，且需要同时发送数据给 CPU2 和 LLC 内存控制器，当 CPU1 再次执行 `a = 2` 时，又需要从状态 **S -> M**，且伴随着数据从 LLC 拷贝到 CPU1 的缓存中，这其中伴随着多次的数据传输，如下图

![image-20240615110754151](/assets/coherence/image-20240615110754151.png)

于是引入 **Owner state** 优化这种情况，简而言之，就是优化了当 block 处于 M 状态时，收到来自其他处理器GetS 请求时的情况，其状态机如下：

<img src="/assets/coherence/image-20240615112317883.png" alt="image-20240615112317883" style="zoom:33%;" />

有了 Owner state，那么之前的例子，就如下图所示：

![msio](/assets/coherence/msio.gif)

可以看到，如此一来，原先需要两次的数据传输，现在就只需要一次就可以了，并且也不需要 LLC 控制器进行任何的处理。

### MOSIE protocol

对于现如今的大多数处理器来说，比如 X86 处理器，都会使用 MOSIE 协议来实现 cache coherence，来最大限度的优化内部互联网络的使用率，当然，其状态机也就更复杂，如下图：

<img src="/assets/coherence/image-20240615113450984.png" alt="image-20240615113450984" style="zoom:33%;" />

## reference

1. [*A Primer on Memory Consistency and Cache Coherence*](https://pages.cs.wisc.edu/~markhill/papers/primer2020_2nd_edition.pdf)
2. [*A Basic Snooping-Based Multi-Processor Implementation*](https://www.cs.cmu.edu/afs/cs/academic/class/15418-s18/www/lectures/12_snoopimpl.pdf)
3. [*Exploring How Cache Coherency Accelerates Heterogeneous Compute*](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/exploring-how-cache-coherency-accelerates-heterogeneous-compute)
4. [*Cache coherency protocols (examples)*](https://en.wikipedia.org/wiki/Cache_coherency_protocols_(examples))
