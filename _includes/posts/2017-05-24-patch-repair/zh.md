> 2015年时和一个老师讨论过一个题目，大意是使用新版本系统的补丁自动化修复老本系统的bug，这个大背景是老的系统由于官方原因不再提供维护和补丁，但是其和新版本的内核有很多的共用。下面的资料就是在做可行性分析的时候整理的，同时还有部分实验结果。
>
> 以下只用作自我反思，请勿转载。

为什么选择Windows？不像其它厂商，微软会较为有规律地发布安全补丁，而且补丁通常都只修补一个应用程序的一个或几个漏洞,所以补丁较为容易分析。
ps:实际上就是个噱头，因为Windows不开源，且关注度高，如果分析开源平台的贡献就没那么大了。
在分析方案可行性之前我们首先需要了解Windows补丁的结构，才能确定是否可以通过分析补丁定位漏洞并自动修复。
下图是从微软security官网上爬去下来的数据绘制成的图片，纵坐标时不同版本windows在各年度发布的补丁数量，此图最初时是为了证明这项工作的意义。

![bug-years](/img/in-post/post-patch-repair/bug-years.png)

## 补丁结构静态分析

Windows补丁的扩展名通常.msu，其本质上就是一个压缩包，压缩包里面包含了四个元素，可以用Wusa解析该文件：
- Windows Update metadata, which describes the update package
- One or more .CAB files that store the update data for the package
- An .XML file which describes the contents of the MSU file
- A properties file that is read by Wusa.exe

![package](/img/in-post/post-patch-repair/update-metadata.png)
我对这四类文件进行了检查分析：
properties.txt主要是被用来判断是否适用于当前系统版本，同时对系统进行自检，看对应问题点是否已经处理。
.xml文件中的action定义了该msu指定系统对update package文件的操作，location定义了补丁安装文件“x86.cab”在msu释放后所在的位置和名称。
![package](/img/in-post/post-patch-repair/wusa.png)
如上图所示，官方说WSUSSCAN.cab是微软用来同步更新文件的东西，那么具体保存的是什么呢，我对三个补丁中解压出的wsussan.cab文件使用二进制文件比较工具进行比对，只有package.xml文件中的package-id,creationdate,updateid等信息发生了改变，可以判断官方会在为每个补丁文件更新WSUSSCAN.cab，但是除了上面信息外，其他内部描述都不变。
![package](/img/in-post/post-patch-repair/kbxxx.png)
KBXXX.cab是更新数据所在的位置，其中manifest.xml是压缩包中文件的描述信息，说明了补丁压缩包中包含的文件信息，安装信息不在这个文件中。每个package_xxxx.cat文件对应了一个.mum文件，该文件的作用是
a file with the extension .mum is an update file that contains information that windows update uses to execute the patch on your computer.
总的来说，KB_XXX.cab中描述了补丁替换的目的文件所在的绝对路径，其所涉及到的注册表等重要信息。

通过静态分析补丁文件可以得到的信息是，其中的静态描述性文件，在我们知道目标操作系统版本的情况下是可以很容易的构造出来的，那么，现在的关键问题就是，不同Windows系统版本的内核代码共用多吗，比如Win7和XP。
当然分析这个关键问题不能通过静态对比内核文件，主要是工作量太大，我选取了相对简单的方法，我们选取一定数量的Win7和XP的补丁，对比其中的二进制文件，看修补的函数差异情况，通过这种统计就能够知道该想法的可行性了。

## 补丁文件比较

#### 论文
《一种利用ms补丁信息分析windows软件漏洞的方法》
- 分析方法：模式比较
- 思想：由大范围逐行代码的比较，转为大范围的函数同构性比较，加上小范围的逐行代码比较，以分析被修改的函数语义的变化。
- 方法：通过IDA得到补丁安装前后两个pe文件的反汇编文本，并分别以函数位单位对代码进行划分。然后将函数意义配对，形成PE文件之间的比较列表，列表的每个数据元素包括一对函数信息，如函数入口地址，长度，参数总长度等。形成每队函数的两个模式图，函数中每条汇编指令都是该函数模式图的一个节点，通过函数比较算法对函数的指令模式图进行相似性比较，得到同构性链表。对比较结果进行分析得到问题代码的位置。
函数模式图：以图的形式描述函数中汇编指令流程，包括汇编指令。
- 工具：win32graph in ida
- 函数比较：指令比较的关系包括：相似，接近，不相关（一条指令在比较的时候是否可忽略），得到函数的同构性链表。
- 模式图染色：将函数的差异性通过色彩标识体现到链表图中，这一步主要是为了方便人工识别。
- 总结：本文的核心思想就是比对安装补丁前文pe文件的差异，只不过在比较方法上有优化。

## 二进制文件比较

从最早的BMAT至今已经有十年之久，直到最近十年二进制比较才被人们广泛使用。除了昂贵的“bindiff”，我们目前有多个开源的工具可以用于补丁分析。这里我们大概介绍一下历来有哪些二进制比较的理论和工具。

#### [BMAT(1999)](https://www.jilp.org/vol2/v2paper2.pdf)
这种方法严重地依赖于符号文件，它主要是用于微软的二进制文件，因为这些文件都是有符号文件下载的。在基于名称匹配的基础上，当所有的函数都已经被匹配以后，它会对函数内的基本块进行基于哈希的比较。它从汇编指令生成64bit的哈希值，并对指令和操作数进行了抽象分级。
这篇文章并没有集中于安全补丁的分析。它主要是介绍如何通过校检值来比较块。如果代码优化参数被改变，那么生成的基本块也会被改变，但是一般厂商并不会改变这些优化参数，所以基本块都会保持原来的指令顺序不变。所以对块计算校检值并用于比较比比较整个所有的指令要好一些。文章中还提到了五种不同级别的块校检值计算方法。他们将这五种级别称成为“matching fuzziness” 级别。
级别越低，用于计算校检值的信息就越多，0 级基本上将所有信息都用于计算校检值，包括寄存器、块地址、操作数、opcode 指令等等，而第五级只使用 opcode 指令生成校检值。

#### [Bindiff](https://static.googleusercontent.com/media/www.zynamics.com/en//downloads/bindiffsstic05-1.pdf)
Halvar在Blackhat 2004就这个议题进行了演讲。他提出使用函数的指纹(fingerprints)来进行比较。而且他基于这个独特而简单的思路开发了著名的 bindiff。Bindiff使用节点的个数、边界数以及调用数作为特征，生成一个函数的签名，然后使用它进行比较。该工具还使用了基于函数 CG（call graph）的同构比较。[最近好像又除了新版本？](https://blog.csdn.net/fjh658/article/details/77646526)Angr也出了对应的工具。
Bindiff的问题是该算法严重依赖于二进制文件的CFG生成，就如他所说的，从二进制文件生成CFG不是件容易的事。生成CFG本身就是一个富有挑战性的工作。任何矛盾的CFG都会导致整个分析失败。你可以忽略CFG识别错误，但是这就意味着许多错误的结果。而且你可能会错过非常重要的部分。
而且这种算法无法分析没有结构性变化的补丁，例如仅仅是改变了某个常量或者换了寄存器。Patchdiff2(2008)和Halvar的方法类似。

#### 基于同构分析的二进制比较 (2004)
Todd Sabin提出了基于同构分析的二进制比较方法。它基于指令图形的同形匹配。它不再对函数进行分割，而是直接将整个函数结构进行同构匹配。有意思的是它比较指令而不是基本块。他声称性能还不错可以用于使用，但是从未公布工具。

#### [DarunGrim2](http://www.darungrim.org/)
关于DarunGrim使用到的算法，更多详细的内容可以参考这个[文档](/img/in-post/post-patch-repair/patch.pdf)。
- 符号名称匹配:DarunGrim2 也像其他程序一样使用符号名称匹配，这是二进制过程匹配的最基本的方法。符号名称可能是导出的名称也可能是符号文件提供的名称。一般来说符号文件是不公开提供的， 但是微软一直为所有补丁提供符号文件。这对于二进制比较和漏洞挖掘有非常重要的作用。
- 指纹哈希:这种方法非常简单，不需要对二进制文件非常精确地分析就可以实现。指纹哈希采用指令序列作为特征值。
一般说来，指纹可能存在多重含义。这里指纹是用于表示一个基本块的数据。对于每一个基本块，DarunGrim2 都从中读取所有的字节并作为key 存储在哈希表中。我们可以称之为块的指纹。指纹的大小随着字节的不同而不同。有很多其他方法也可以生成指纹。
指纹匹配可以非常有效地匹配基本块。基本块是二进制比较的最基本的分析元素，一个基本块可能含有多个引用。DarunGrim2为比较的两个二进制文件分别建立指纹哈希表，然后将每一项都进行匹配。不像flirt那种传统的函数指纹，DarunGrim2用得是抽象的字节作为基本块的指纹。所以即使函数中有些基本块被修改了，根据其它匹配的块依然可以正确地将函数匹配。为了快速地匹配海量的指纹哈希，我们把生成的指纹串存在哈希表中然后进行匹配。有很多方式可以生成基本块的指纹，你也可以尝试其他的指纹生成方式，匹配的时候略有不同。
最简单的生成指纹的方式就是使用指令和操作数。因为基本块的指纹生成是至关重要的部分， 所以还有一些问题需要考虑，例如编译和链接选项产生的差异。我们会忽略内存和立即数，因为改变源代码的时候很容易导致这种差异。我们也会选择性地忽略一些寄存器差异。
- 克服顺序依赖的缺陷:连续字串的指纹最大的缺陷就是顺序依赖。实际上这个问题普遍存在于各种算法当中。所以为了解决这个问题我们提供了选项，在生成指纹之前对指令顺序进行整理。这个过程只要判断指令的互相依赖关系即可,然后将指令按照升序排列,互相依赖的指令必须维持他们原有的顺 序。优化指令序列对于编译器指令顺序优化过后的代码尤为重要。
- 减少哈希碰撞:一个二进制文件中会有不少短的基本块，这些基本块很容易重复。这就会导致哈希重复，我们可以直接抛弃这些重复的基本块。但是我们有更好的处理方法，就是将一个基本块和连接它的下一个基本块进行关联。这样重复的哈希值就很少了。
- 检测函数匹配对:在基于哈希表的匹配完成以后，我们必须对函数中匹配的基本块进行统计。如果一个过程匹配多个过程，那我们必须进行选择。我们采用的方法很简单，就是利用匹配的基本块计数，数值最高的作为匹配项，如果有多个同样数值的，那么会随机选择一项。一旦某一项被选择，那么未被选择的块中的基本块关联将被取消。其实这个抉择过程很大程度上依赖于哈希的配对。在实际使用中，这样的哈希配对方法有很好的性能和不错的效果。
- 函数中的匹配块:如果一个过程被选择配对了。那么其中的块也会重新计算哈希表进行配对。这就是函数中的二次匹配。和之前的方法一样，通过关联上下块，重复的哈希值依然会很少。
- 基于结构的分析: 基本块同构,在进行完函数配对并将其中的块一一配对以后，我们将会进行基于结构的分析,这种方法和BMAT 的方法比较类似。从已经匹配的节点，去匹配他们的子节点。

## 实验

#### 实验范围
在微软的官网上筛选了从2010~2012年的近295个补丁，跨度从winxp sp3到win7 sp2，基于分析过程中观察到的实验结果及匹配失败原因的猜想筛选出43个补丁进行深度分析。

#### 实验方法
对winxp，win7的补丁结构进行分析后，发现两者差别较大，所以选择直接对安装完成后被修改的文件进行代码比对，首先做两组对比，进行初步筛选：
- 分别对各版本系统打补丁前后的文件进行比对；
- 对不同版本系统打补丁后的文件进行比对；
上面的比对包括使用用函数名称和长度进行匹配，完成筛选后进行深度分析，这里使用了patchdiff2和darungrim进行分析，patchdiff2速度快，但匹配结果不准确，所以对可能匹配的结果需要darungrim进行再次验证。

#### 实验结果
其中完全匹配的有2个补丁，其中两个匹配的例子如下图所示。
1. ms10-81：
只有SBGetText函数有改动：
![package](/img/in-post/post-patch-repair/1.jpg)
xp的变化点如下图所示（图中黄色及灰色部分代码）：
![package](/img/in-post/post-patch-repair/2.jpg)
win7的变化点如下：
![package](/img/in-post/post-patch-repair/3.jpg)

2. ms12-082：
winxp sp3和win7 sp2的补丁中都只有一个函数有改动：
![package](/img/in-post/post-patch-repair/4.jpg)
winxp：
![package](/img/in-post/post-patch-repair/5.jpg)
win7：
![package](/img/in-post/post-patch-repair/6.jpg)

#### 实验结果分析
对于实验过程中补丁匹配数量少的问题，我认为主要有以下几点原因：
- 内核升级过程中动态链接库内升级增添了新的函数，winxp中不存在，这种情况下不可能匹配。
- 内核升级导致原来一个dll文件分裂成多个文件，内部函数结构必然发生变化。造成无法匹配。
- 微软内部版本迭代可能会比较细，每次修改的点也比较少，但是release版本号跨度很大，每次升级会修改过多的函数，且函数的修改范围很大。同时由于win7和xp的补丁发布已经不同步，内部的函数迭代速度差很多，也会导致修改的点会很多。
- 一个补丁针对一个文件中的多个漏洞，也会造成修改的函数过多>20，很难进行匹配分析。

#### 实验存在的问题
微软内部版本迭代可能会比较细，每次修改的点也比较少，但是release版本号跨度很大，每次升级会修改过多的函数，且函数的修改范围很大，这也可能是导致匹配结果不好的主要原因？这一点不太好验证。

## 参考文献

#### 补丁原理： 
以下文献是对（热）补丁原理的介绍性资料
- [Understanding Software Patching](http://queue.acm.org/detail.cfm?id=1053343)
- Dynamic software updating
 
#### windows patch格式：
因为windows补丁文件的格式并未公开，所以下面的资料都只是和补丁压缩文件格式等内容相关的资料。
- Persist it: using and abusing Microsoft’s fix it patch
- [Using Catalog Files](https://msdn.microsoft.com/en-us/library/aa741204(v=vs.85).aspx)
- [Inter-delta dependent containers for content delivery](https://www.google.com/patents/US20070260653)
- [Catalog Files and Digital Signatures](https://msdn.microsoft.com/en-us/library/windows/hardware/ff537872(v=vs.85).aspx)
- [Using Binary Delta Compression (BDC) Technology to Update Windows Operating Systems](http://www.microsoft.com/en-us/download/details.aspx?id=1562)
 
#### 补丁安全性测试：
- KATCH：High-Coverage Testing of Software Patches,
使用符号执行对Patch代码进行高覆盖率测试，需要对源码进行比对分析，对Patch中新添加的可执行代码进行完整的覆盖测试。
- How do fixes become bugs,
利用Coverage和Disruption指标对Patch进行评价测试
  -- Coverage：对所有触发bug的输入是否都能正常处理。
  -- Disruption：Patch是否引入了原始程序预期执行逻辑之外的逻辑。
- Has the bug really been fixed, 本文针对的是操作系统中的Patch进行测试，主要对Patch编写不当可能引发的错误（并发等问题）在四个操作系统中进行了调研统计，并针对每种类型的错误出现的原因和修改意见做出了说明。
 
#### 补丁修复模式类：
1). Toward an understanding of bug fix patterns

2). Automatically patching errors in deployed software
 
