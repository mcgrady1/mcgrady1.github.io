---
layout:     post
title:      ""
title:      "What is MicroX"
subtitle:   "Paper Summary"
date:       2017-03-19 12:00:00
author:     "Mcgrady"
header-img: "img/in-post/post-microx/gandam.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Symbolic Execution
    - Virtual Execution
    - Testing
    - Paper Summary
---

> MicroX是我在做Memroy Fuzzing时主要参考的几篇文章之一，这里是我对这篇文章主要思想的之前的整理。
>
> Micro execution is the ability to execute any code fragment without a user-provided test driver or input data. The user simply identifies a function or code location in an exe or dll. A runtime Virtual Machine (VM) customized for testing purposes then starts executing the code at that location, catches all memory operations before they occur, allocates memory on-the-fly in order to perform those read/write memory operations, and provides input values according to a customizable memory policy, which defines what read memory accesses should be treated as inputs.
>
> 以下只用作自我反思，请勿转载。

## what is *micro execution*?
"The user selects a function of code location in any executable file, such as an exe or a dll on a Windows machine. A special runtime Virtual Machine (VM) customized for testing purposes then starts executing the code at that location. The VM hijacks all memory operations, and provides input values according to a customizable memory policy."

VM劫持所有的对内存的操作，并根据用户定制的内存策略来提供函数输入值。上面还提供了一个示例策略，这个策略描述的意思对未初始化的函数参数可以使用任意随机值，如果地址值也是输入参数并通过随机生成取得的，则其地址内的值也随机生成。

"An input is defined as any value read from an uninitialized function argument, or from a dereferenceof a previous input used as an address (recursive definition)."

假设用户想微执行下面的这段代码（选择随机测试生成模式），首先处理函数内的第一条指令，VM检测到程序想读一个4字节的值作为输入参数（假设运行在32位机器上），因此随机生成一个4字节数字返回给程序（怎样完成的会在下一个小结进行描述）。接下来执行第二行指令，检测到要使用输入参数取得1个字节的值，因此根据之前制定的策略，因为p也是一个输入参数，所以随机生成一个1字节数值并返回给程序。指令执行完成后终止micro过程。

```c++
void foo(char *p) { // p is a 4-bytes input

  char v = *p; // *p is a 1-byte input

  return;
}
```

总结一下，micro execution动态监测到函数的两个输入值，并通过随机生成的方式返回给函数。使用这种方式，micro不需要构造任意的测试驱动或者input就可以直接测试程序，测试人员不需要了解内存的构造，只需要指定测试起始指令就可以了。

## microX
该工具支持二进制程序的分析，因为函数起始的指令是有特点的。比如通过下面的指令块可以看到，函数的起始首先进行push和mov。

```asm
[...]
1: push ebp ; foo starts here
2: mov ebp, esp
3: push ecx
4: mov eax, DWORD PTR [ebp+8] ; p
5: mov cl, BYTE PTR [eax] ; *p
6: mov BYTE PTR [ebp-1], cl ; v
7: mov esp, ebp
8: pop ebp
9: ret 0
[...]
```
最重要的部分是，第4行中，[ebp+8]是间接寻址，会对内存进行访问，且根据栈的结构，这里应该是函数的第一个输入参数，如果有第二个，则应该在ebp+12处。因为没有给foo函数任何的输入参数，所以[ebp+8]是没有被分配和初始化的。因此microX在一个新的地址x重新分配一个对程序不可见的外部内存，接着，microX将ebp+8和x两个内存联系起来，将ebp+8的地址值用x的地址值进行替换，并将4字节的的参数值存放在[x]中。

```
1: initEIP is 72B51005
2: initEBP is 001EF988
3: Read Mem Access at address 001EF990 of 4 bytes
4: Initializing 4 input bytes:
5: [0]=78 [1]=14 [2]=20 [3]=00
6: Adding 00201478 to list of known addresses
7: SetGuestEffectiveAddress returned 00201440
8: Read Mem Access at address 00201478 of 1 bytes
9: Initializing 1 input bytes: [0]=29
10: SetGuestEffectiveAddress returned 0020C490
11: Write Mem Access at address 001EF987 of 1 bytes
12: SetGuestEffectiveAddress returned 001EF987
13: END: ExitProcess is called
14: ***** External Memory Stats: *****
15: Number of Mem Accesses: 2 (2 Reads, 0 Writes)
16: Number of Addresses: 2 (total 5 bytes)
17: Number of Inputs: 2 (total 5 bytes)
18: ***** Native Memory Stats: *****
19: Number of Module Accesses: 0 (0 Reads, 0 Writes)
20: Number of Other Accesses: 1 (0 Reads, 1 Writes)
21: ***** General Stats: *****
22: Number of Unique Instructions After Start: 9
23: Number of Warnings: 0
24: Number of Errors: 0
```
上面是microX生成的内存访问报告，可以看到，ebp的初始值为 0x001ef988，eip的初始值为0x72b51005
对上面的内容我存在一个疑问，这两个初始值是怎样得到的。这里先忽略这个问题。第一次对内存的访问地址为 0x001ef990，即ebp+8，如上面所说，当使用随机生成模式的时候，microX会返回一个随机生成的值00201478，此时还不知道这个返回值会作为一个指针使用，为了预防，将这个值记录到known input address 列表里面，这四个字节被储存在一个4字节的buffer中，在本例中被定为到00201440这个地址处，从此刻起，所有对0x001ef990处的访问都会被映射到0x00201440，该地址是程序不可见的，而被microX控制的外部地址，因为0x00201440从没有在程序中的任何寄存器中出现过。

第5行中，micro检测到之前的随机生成值被当做地址使用（因为已经放入known list中），对一个输入的间接引用同样应该被看成一个输入值，所以这里microX重新分配了一个以自己的buffer，地址为0020c490，初始化该地址并生成一个一字节的随机数29，并将0020c490和00201478建立映射。下面执行第6行代码，可以看到，操作访问了ebp-1，因为ebp以下的区域不会存在输入参数，而且这是一个写操作，所以不用定义新的内存。所以将0x001ef987映射到其自己。

## MicroX implementation

#### Memory Access

MicroX重定向所有的内存访问操作是怎样实现的呢？首先动态的解释每条将执行的x86指令，并将内存操做相关的指令拆分成微指令序列集合，将其中的内存访问地址进行替换。举例说，对于x86指令
```asm
mov eax，[ecx]
```
该指令的含义是：
- 使用eax寄存器中的值作为地址
- 获取[eax]地址中的值
- 将该值储存到寄存器eax中
这些操作被microX重写成型如西面代码的新指令

```
...
GenerateEffectiveAddress
...
PREMemoryAccessCallBack
...
mov eax, [EffectiveAddress]
...
```
Generate是一个x86中的宏定义（应该类似于c语言，只是简单的替换操作），计算哪些地址被当前的指令所访问。然后将这些地址存储到一个变量Effectiveaddress中，在 Effectiveaddress被访问之前需要有一个回调函数PREMemoryAccess，该回调函数检查当前的内存访问策略来决定当前的 Effectiveaddress如何访问
- 不处理
- 用一个新的地址进行替换，如前面的foo函数一样，如果此处为输入参数就需要对地址进行替换。microx会自动在扩展内存中分配区域，并保存一个从effective地址到扩展内存地址的映射。并将effective的值替换成external address的值。被测试代码是没有感知到该替换操作的。
接下来，x86指令执行mov eax，[effectiveaddress]的操作，该操作很好的保证了指令的语义信息不被改变，也就是说对进程的eflags寄存器是没有影响的。

想强调的是，执行中的指令是不会知道external memory的存在的，例如在上面的例子中，ecx中的值并没有被替换成external mamory中的值，因为，microX并不知道ecx中的值的来源，和程序在后面需要使用ecx中的值做什么操作，改变ecx中的值可能有负面的影响。

#### External Memory Management

其维护了一个从程序可见的地址到扩展内存地址的映射，这个映射保证了读写的一致性。当一个输入值q被写入扩展地址a，则每次要读取q都回去访问a，并返回相同的值。伪代码如下：

```c++
1: int r; // Heap_Address_Range; default 250
2: int r_EBP; // EBP_Address_Range; default 100
3: int InitEBP; // initial value of EBP
4: list_of_ADDR KnownInputAddresses;
5: bool IsInputAddress(ADDR a) {
6: if ((InitEBP <= a) && (a < (InitEBP + r_EBP)))
7: return true;
8: for any x in KnownInputAddresses {
9: if (((x >= a) && ((x - a) < r))
10: || ((x < a) && ((a - x) < r)))
11: return true;
12: }
13: return false;
14: }
15: ADDR PREMemoryAccessCallback
16: (ADDR a, int size, bool isRead)
17: {
18: a’ = ExternalMemory(a);
19: if (a’ is defined) return a’;
20: if (!IsInputAddress(a)) return a;
21: Add ADDR [a,a+size-1] to ExternalMemory;
22: a’ = ExternalMemory(a);
23: if (isRead) {
24: initialize the values at ADDR [a,a+size-1]
25: in ExternalMemory;
26: if (size == 4)
27: { add the 4-bytes value at [a,a+3]
28: to KnownInputAddresses; }
29: }
30: return a’;
31: }
```

