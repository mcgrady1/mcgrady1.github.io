> 2015年时和一个老师讨论过一个题目，大意是使用新版本系统的补丁自动化修复老本系统的bug，这个大背景是老的系统由于官方原因不再提供维护和补丁，但是其和新版本的内核有很多的共用。下面的资料就是在做可行性分析的时候整理的，同时还有部分实验结果。
>
> 以下只用作自我反思，请勿转载。

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
如上图所示，官方说WSUSSCAN.cab是微软用来同步更新文件的东西，那么具体保存的是什么呢，我对三个补丁中解压出的wsussan.cab文件使用二进制文件比较工具进行比对，只有package.xml文件中的package-id,creationdate,updateid等信息发生了改变，可以判断这个文件会在补丁生成的时候重新生成一次，但是除了上面信息外，其他内部描述都不变。
![package](/img/in-post/post-patch-repair/kbxxx.png)
KBXXX.cab是更新数据所在的位置，其中manifest.xml是压缩包中文件的描述信息，说明了补丁压缩包中包含的文件信息，安装信息不在这个文件中。每个package_xxxx.cat文件对应了一个.mum文件，该文件的作用是
a file with the extension .mum is an update file that contains information that windows update uses to execute the patch on your computer.
总的来说，KB_XXX.cab中描述了补丁替换的目的文件所在的绝对路径，其所涉及到的注册表等重要信息。

通过静态分析补丁文件可以得到的信息是，其中的静态描述性文件，在我们知道目标操作系统版本的情况下是可以很容易的构造出来的，那么，现在的关键问题就是，不同Windows系统版本的内核代码共用多吗，比如Win7和XP。
当然分析这个关键问题不能通过静态对比内核文件，主要是工作量太大，我选取了相对简单的方法，我们选取一定数量的Win7和XP的补丁，对比其中的二进制文件，看修补的函数差异情况，通过这种统计就能够知道该想法的可行性了。

## 补丁文件比较

关于补丁安装前后文件的对比方法找到一片文章《一种利用ms补丁信息分析windows软件漏洞的方法》，
分析方法：模式比较
思想：由大范围逐行代码的比较，转为大范围的函数同构性比较，加上小范围的逐行代码比较，以分析被修改的函数语义的变化。
方法：通过IDA得到补丁安装前后两个pe文件的反汇编文本，并分别以函数位单位对代码进行划分。然后将函数意义配对，形成PE文件之间的比较列表，列表的每个数据元素包括一对函数信息，如函数入口地址，长度，参数总长度等。形成每队函数的两个模式图，函数中每条汇编指令都是该函数模式图的一个节点，通过函数比较算法对函数的指令模式图进行相似性比较，得到同构性链表。对比较结果进行分析得到问题代码的位置。
函数模式图：以图的形式描述函数中汇编指令流程，包括汇编指令。
工具：win32graph in ida
函数比较：指令比较的关系包括：相似，接近，不相关（一条指令在比较的时候是否可忽略），得到函数的同构性链表。
模式图染色：将函数的差异性通过色彩标识体现到链表图中，这一步主要是为了方便人工识别。
总结：本文的核心思想就是比对安装补丁前文pe文件的差异，只不过在比较方法上有优化。

说了一堆，其实我可以直接用更简单的办法：将两个pe文件直接通过compare或者bindiff找到差异代码，再用ida定位到具体的函数，但是定位到了具体的函数，还是需要知道两个函数在处理流程上的变化,这样才能自己捕获到问题点和补丁代码的功能。总之我们现在的目标首先是看一下不同系统版本补丁修复前后差异点的相似度，不废话，直接上结果。

这里我们使用了patchdiff2，ida pro的一个插件，不知道目前是否还在更新，其基本原理是根据名称+crc的方式进行对比和匹配，但因为编译后信息的缺失，其实还是不太准确，但相比于我自己写的名称夹函数长度的匹配方式还是好一点。
 
