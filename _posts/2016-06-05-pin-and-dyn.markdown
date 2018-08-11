---
layout:     post
title:      "Pin vs Dynamorio"
subtitle:   "Project Summary"
date:       2016-06-05 12:00:00
author:     "Mcgrady"
header-img: "img/in-post/post-pin-dyn/tracy-mcgrady-vince-carter.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Instrumentation
    - Code Cache
---


> Pin和Dynamorio都是我们在进行程序动态分析时经常会使用到的插装平台，这两者的区别是什么，Dynamorio的效率为什么比Pin高很多，通过这篇文章的分析来探索其中的答案。
>
> Pin是Intel自己维护的平台，文章发表在PLDI’05，Dynamorio现在由Google的一个小团队在维护，相比Pin，其开源性更好，从底层架构到所有插件用户都可以进行修改。
>
> 以下内容只供自己回顾使用，请勿转载。

# Pin

Pin是由Intel开发的一个支持IA32(32bit x86), EM64T(64-bit x86), Itanium和ARM(Intel声称)四种平台的插桩工具。Pin在运行时对于其所监控的程序来说是完全透明的（上帝视角），使用动态编译（JIT）的方法，结合内联、寄存器重分配、活跃性分析以及插桩指令的一些优化性的调度等策略，对程序进行动态监控。

## 0x0 Instrumentation with Pin

Pin能够看到进程所有的上下文，包括内存、寄存器和控制流。PIN开发的函数可分为Instrumentation routine和Analysis routine。详情请移步 [pin入门资料](https://software.intel.com/sites/landingpage/pintool/docs/58423/Pin/html/)

在Pin的开发过程中，将所有的程序分为3类：Pin, Pintool, APP。

- Pin即Intel官方提供的一个已经编译好的可执行程序
- Pintool是开发者基于Pin官方的各种API开发出来的辅助工具，也即我们的主要debug战场，以及各种链接进来的需要与pin进行通信的库。
- APP即为我们要用Pin来监控的程序或进程（可在Pin参数中指定某个进程的pid，通过attach的方式来监控进程）。
- Pin，PinTool和APP会使用相同的地址空间，ps：这里说的不一定对，Pin和APP实际是两个进程，Pin.exe启动后，通过ptrace加在APP，然后调用injector将JIT，Pintool等都加载到APP的空间，所以调试权限还是在父进程Pin手中。在linux下，在pin的执行过程中，有3份glibc的拷贝。pin与app不共享任何数据，避免重入问题发生。何谓重入问题，作者举了一个例子，假设APP要调用glibc中的某个函数，此时控制权会交给JIT，此时JIT可能也需要使用这个函数， 则JIT也要进入这个函数，而此时该函数的调用和控制权应该在APP，这样就会产生一些问题。

Pin拦截APP（进程）的第一条指令（当前指令），并生成从这条指令起的线性代码序列，然后将控制转到生成的序列上去，当遇到分支跳转时，pin重新获得控制权限。转换和插桩后的代码被放在一个代码缓存中，以便以后再次执行时提高效率。由于PIN良好的封装，使得我们能够跨体系结构进行开发，不论底层的机器指令集是RISC, CISC还是VLIW(very long instruction word),而无需更改底层的代码，非常的方便友好。

## Design and Implementation

#### System Overview

从高层来看，PIN由VM, 代码缓存（code cache）,以及一个由PINTOOL调用的插庄API组成。VM由JIT，一个模拟器和一个调度器（dispatcher）组成，在pin获得程序的控制权限后，VM负责协调所有的组件来执行程序。

![overview](http://loccs.sjtu.edu.cn/gossip/images/2015-03-04/pin_overview.jpg)

- JIT和程序的插桩代码都是由调度器来启动。编译过的代码存储于代码缓存中，只有代码缓存中的代码才是可执行的，这也意味着在用pin对程序进行动态插桩的过程中，即使程序原来的代码是可执行的，也将变得不可执行。每次在代码缓存和VM之间进行切换时都需要存储/恢复寄存器现场（所以可以看到开销是非常大的）。
- 模拟器拦截不能被直接执行的指令(如syscall)，pin只能捕获用户层面的代码。

#### Injecting Pin

pin的插入器使用ptrace来获取app的控制权限和程序运行时的上下文。这里插播以下ptrace及gdb的原理，gdb就是在ptrace的基础上做的调试工具。

##### ptrace
建立调试关系的方法：
- fork: 利用fork+execve执行被测试的程序，子进程在执行execve之前调用ptrace(PTRACE_TRACEME)，建立了与父进程(debugger)的跟踪关系。
- attach: debugger可以调用ptrace(PTRACE_ATTACH，pid,...)，建立自己与进程号为pid的进程间的跟踪关系。即利用PTRACE_ATTACH，使自己变成被调试程序的父进程(用ps可以看到)。用attach建立起来的跟踪关系，可以调用ptrace(PTRACE_DETACH，pid,...)来解除。注意attach进程时的权限问题，如一个非root权限的进程是不能attach到一个root进程上的。

断点原理：
- 断点的实现原理，是在指定的位置插入断点指令，当被调试的程序运行到断点的时候，产生SIGTRAP信号。该信号被gdb捕获并进行断点命中判定，当gdb判断出这次SIGTRAP是断点命中之后就会转入等待用户输入进行下一步处理，否则继续。 
- 断点的设置原理: 在程序中设置断点，就是先将该位置的原来的指令保存，然后向该位置写入int 3。当执行到int 3的时候，发生软中断，内核会给子进程发出SIGTRAP信号，当然这个信号会被转发给父进程。然后用保存的指令替换int3,等待恢复运行。
- 断点命中判定:gdb把所有的断点位置都存放在一个链表中，命中判定即把被调试程序当前停止的位置和链表中的断点位置进行比较，看是断点产生的信号，还是无关信号。
- 条件断点的判定:原理同3)，只是恢复断点处的指令后，再多加一步条件判断。若表达式为真，则触发断点。由于需要判断一次，因此加入条件断点后，不管有没有触发到条件断点，都会影响性能。在x86平台，某些硬件支持硬件断点，在条件断点处不插入int    3，而是插入一个其他指令，当程序走到这个地址的时候，不发出int 3信号，而是先去比较一下特定寄存器和某个地址的内容，再决定是否发送int 3。因此，当你的断点的位置会被程序频繁地“路过”时，尽量使用硬件断点，会对提高性能有帮助。

单步跟踪原理：
- 因为ptrace本身支持单步功能，调用ptrace(PTRACE_SINGLESTEP，pid,...)即可。

injector将pin binary加载到app的地址空间中，开始运行，在自身初始化结束后，pin将Pintool加载到地址空间中并让其运行，在pintool初始化完成后，向pin发出请求，启动app，pin创建初始化上下文并在入口点开始jitting app(或进程)。

DynamoRIO依赖于环境变量LD_PRELOAD，迫使动态加载器先将这个共享库加载到地址空间中。pin的方法相对于此，好处有三： 1. LD_PRELOAD无法在静态链接bin中正常工作 2. 加载额外共享库可能将APP的所有共享库（以及一些动态链接库）移动到高地址，而pin尽量使其保留在原来的地址 3. 在共享库加载器部分加载后，这些插桩工具才能获得app的控制权，而pin可插在程序的第一条指令。4. DynamoRio的插件和其自身是不是用glibc的，所有的操作都通过直接调用系统函数完成。

DynamoRio不同于Pin，其从一开始就是在APP的空间中运行的，其并没有自己独立的进程，这样从理论上来说是更快的，通过上下文的切换在DynamoRio分析代码和程序代码中跳转。虽然其存在上面的问题，虽然调用DynamoRio的汇编接口写插装代码很麻烦。

#### The JIT Compiler

#####  Basics

pin将一个ISA编译到同一个ISA（无法跨体系结构平台编译），且不需要经过一个中间行驶，编译好的代码存在一个基于软件的code cache上。

- 只有在code cache上的code 是可执行的，原始code不可执行。
- 一次编译一条trace，一个trace是一个指令的线性序列，终结于以下条件
  - 一个无条件跳转（branch, call 或 ret)
  - trace中的条件跳转数量已达到一个预定义的阈值
  - trace中的指令数量已达到一个预定义的阈值

除了最后一个exit，一条trace可能还有多个条件跳转exit，这些exit都分支到一个stub,并将控制权交回给VM，在VM确定目标地址后（预先通过静态的方式无法知道），生成一条新trace（假设在之后未生成过该条trace），并移动到目标trace上继续执行。

#### Trace Linking

为了提高性能，pin将直接从一条trace出口跳转到目标trace，绕过Stub和VM，这个方法称为trace linking.连接一个直接jmp,就好像这个jmp只有一个目标target。这样做的原因在于间接跳转可能有多个目的地址，故需要目标预测机制。ps：这里我有点奇怪，bb的基本定义就是代码中间没有跳转，Pin这样已经改变了bb的含义。

![pin1](http://loccs.sjtu.edu.cn/gossip/images/2015-03-04/pin1.jpg)

- 此处使用jecxz指令是为了避免影响eflag寄存器，若ecx为0则跳转
- 使用一个predicted target链表的结构来储存一个间接跳转的所有可能目标地址。
- 若在这个链表中找到了预期的跳转地址，则直接通过jecxz $matchX来跳转,避免了后续转VM执行
- 若在前述链表中未找到预期跳转地址，则在一个LookupHtab_1中寻找，若仍未找到，则转VM继续执行。

这里的非间接跳转机制与DynamoRIO不同之处有三：
- DynamoRIO中，整个链是一次生成的，并直接将翻译后代码嵌在间接跳转处，因此之后无法再添加新的target addr。而pin可以在程序运行时动态增量式地插入新target(在链表头或者尾部插入)，下一次再遇到这些target就不用去搜索hash table或转vm继续执行了。这里我的理解是，DynamoRio的插装一旦完成，这部分代码的控制权是在APP自身的，所以我们不能对一处代码在不同的时间节点对其进行插装，换句话说，只要代码开始运行你就不能插了，大家可以自己尝试一下就明白了。而PinTool的这部分代码控制权应该还是在Pin的手里，所以可以增量的插装。
- DynamoRio为间接跳转使用的是全局hash table，Pin使用的是局部hash table，在[Hardware Support for Control Transfers in Code Caches @ MICRO-36](http://www.microarch.org/micro36/html/pdf/kim-HardwareSupport.pdf) 中证明过，局部比全局更高效，这里说的局部是什么范围？
- 使用函数克隆技术加速最常见的间接跳转：return. 若一个函数在多处被调用，则给这个函数做多份拷贝，在它的每个调用点都使用一份独一无二的拷贝，这样每个return 就相当于只有一个target(在大多数情况下)，否则一个return由一个target chain，时间开销过大（也即空间换时间）。（在实现中，为每条trace关联一个call stack,每个调用栈都可记录最后4个调用点，并通过hash压缩成一个64-bit 整型数据）.

![pin2](http://loccs.sjtu.edu.cn/gossip/images/2015-03-04/pin2.jpg)

##### Register Re-allocation

在jitting过程中，经常需要使用额外寄存器。不用于通过特殊渠道获取额外寄存器，pin使用线性扫描寄存器来重新分配的方法，对APP和pintool进行寄存器重分配。pin分配器在做interprocedural分配时是唯一的，但在执行时增量发掘流图的过程中，必须一次编译一条trace. 相反，一次能编译一个文件的编译器和字节码JIT，都能一次编译整个方法。

**Register Liveness Analysis** trace exit出的精确reg活跃信息使得寄存器分配更有效。因为pin能在不导致register spill的情况下重用“已经死掉的”寄存器。

没有完整的流图，只能增量式计算活跃信息，在A地址处的trace被编译后，将A作为key，把trace开始时的活跃信息记录在hash table中，如果trace exit是一个静态已知的target,则从hash table中取出活跃信息，以便更精确计算当前trace活跃信息。尽管有时空开销，但是这个方法很有效，能够减少register spill.

**Reconciliation of Register Bindings** 在进行寄存器重分配时，JIT需要保证在trace exit处的寄存器绑定信息与目标trace入口处的绑定信息相同。

![pin3](http://loccs.sjtu.edu.cn/gossip/images/2015-03-04/pin3.jpg)

若target trace未编译，则在编译时使用新的virtual-to-physical寄存器分配信息，若trace已编译且Be != Bt(Be为当前trace exit处reg分配信息，Bt为目标trace入口处reg分配信息)，则加上转换代码。

valgrind采取的则是有些简单低效的做法，在每个基本块末尾，所有的virtual reg都被重新存入内存中。在真实环境下，通常只有1~2个virtual reg不同，而因此pin比valgrind更有效。

在实践中还有一个问题就是，将virtual reg映射到physical reg的补偿代码应该放在何处？是在分支之前还是跳转之后。pin是选择将补偿代码放在分支之前，因为实验数据显示，这将导致更少的唯一绑定，因此减少了编译器的内存开销。

##### Optinmizing Instrumentation Performance

Pin大部分的开销都是在执行Analysis routines上，（ps：对于任何插装工具，这部分都占据了很大的时空开销，之前在给DynamoRio写插件时还遇到一个奇葩的问题，对于REP这种指令，如果你想在每一次循环处都插入分析代码是会出问题的，官方给我的回复时，可能加入分析代码后code cache超过了某些限制，所以会crash）而不是编译的时空开销（包括插入instrumentation rtn）。故对Analysis rtn的优化很重要，无论是调用Analysis rtn的次数还是Analysis rtn本身的复杂性。

PIN的JIT优化使用的是内联技术，能够减少执行开销。没有内联的话，会调用1个bridge routine保存所有寄存器，设置 analysis rtn参数，并调用 a rtn, 通过inline消除bridge，省去了2次call和ret,也不再需要额外存储寄存器。因此需要重命名和重分配寄存器，以管理reg spill。并且，Inlining之后，可以使用很多其他的优化技术，如 a rtn参数的常量折叠等。另外对x86下访问eflag做特别的优化。

在插桩过程中可使用IPOINT_ANYWHERE来优化，这样有更多的机会来优化，比如pin能找到个避免eflag spill的点来插入Analysis rtn。

# DynamoRIO
## 基本架构
下图描述了DynamoRIO设计架构： 
![这里写图片描述](https://img-blog.csdn.net/20160303180240244) 
下图展示了DynamoRIO的各个组件是如何运转的:
![这里写图片描述](https://img-blog.csdn.net/20160303180257765)

## 代码分析及链接
#### 指令缓存（Code cache）
DynamoRIO是一个进程级别的emulation软件，工作在应用和操作系统之间。通过code caching, linking和 trace building提高了emulation的效率。DynamoRIO运行的代码和应用程序本身的代码，通过context switch分开。应用程序代码被拷贝到指令缓存中。 这些缓存中的代码，会像原生代码一样执行，直到遇到一个跳转指令，应用的machine state会被保存，控制转回到DynamoRIO，去寻找跳转指令所在的basic block。(a context switch) 
纯粹的emulation比原生代码执行慢大概300倍，如下图所示： 
![这里写图片描述](https://img-blog.csdn.net/20160303180354109) 
DynamoRIO通过引进code cache机制，可以把这个负荷降低到25倍左右： 
![这里写图片描述](https://img-blog.csdn.net/20160303180403136)

#### 基本块（Basic block）
dynamoRIO的basic block和编译器产生的basic block不太一样。因为dynamoRIO工作在运行时，考虑效率，没有对代码进行太多的分析。

#### 链接（Linking）
DynamoRio把每个basic block都拷贝到一个code cache中，进行原生运行，这样很大地减少了解释运行的开销。然而，我们还是需要对每一个跳转指令进行解释，再返回到DynamoRIO中去寻找这个目标指令。 
** 直接链接（Direct linking）** 
如果这个目标指令已经存在于code cache中，而且被直接跳转指令所指，DynamoRIO可以直接跳转到code cache中的目标指令，避免这次context switch的开销。 
![这里写图片描述](https://img-blog.csdn.net/20160303180513609) 
** 间接链接（Indirect branch）** 
条件转移指令不能像直接跳转指令一样进行link，因为它的目标不止一个。需要进行判断并查找队形的跳转目标。 
![这里写图片描述](https://img-blog.csdn.net/20160303180522669)

#### 执行流（Traces）
一些经常顺序执行的basic blcoks被组合到一个执行流中，这样就减少了分支，提高了程序的局部性。避免了对一些indirect branch的查找，因为我们已经把indirect brach的目标也放到这个trace里来了。
![这里写图片描述](https://img-blog.csdn.net/20160303180552938)

#### 透明（Transparency）
DynamoRO需要尽量做到透明，也就是说，不去影响应用程序本身的运行。但是由于DynamoRIO和应用程序同时运行在用户空间，所以想要做到透明，是有一些麻烦的。

## 资源使用冲突（Resource Usage Conflicts）

** 库透明（Library Transparency）** 
dynamoRIO在装载应用程序时，dynamoRIO本身的代码也会使用一些库（share libraries）,如果同时应用程序也使用了同样的一个库，可能会造成一些冲突（比如：error code冲突）。 
解决方法就是，DynamoRIO本身不使用库，在linux上直接调用system call，在windows上通过win32 API简介调用system call。 
![这里写图片描述](https://img-blog.csdn.net/20160303180627779) 
** 堆透明（Heap Transparency）** 
DynamoRIO自身分配的堆内存应该和应用程序申请的堆内存区分开来。 
** 输入/输出透明（I/O Transparency）** 
DynamoRIO在做输入输出时，用的是自己写的I/O子程序，用来避免和应用程序的I/O缓冲区冲突。 
** 同步透明（Synchronization Transparency）** 
共享锁的使用也会造成造成DynamoRIO和应用程序的冲突。
** 线程透明（Thread Transparency）** 
为了避免和应用程序的冲突，DynamoRIO不改动程序自身的线程，且尽可能少的创建自己的[线程](http://www.burningcutlery.com/derek/docs/threadshared-CGO06.pdf)。
** 栈透明（Stack Transparency）** 
DynamoRIO选择让应用进程的栈保持不变，它在每个线程中，都创建了一个自己私有的栈。

