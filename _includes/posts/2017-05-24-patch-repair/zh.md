> 2015年时和一个老师讨论过一个题目，大意是使用新版本系统的补丁自动化修复老本系统的bug，这个大背景是老的系统由于官方原因不再提供维护和补丁，但是其和新版本的内核有很多的共用。下面的资料就是在做可行性分析的时候整理的，同时还有部分实验结果。
>
> 以下只用作自我反思，请勿转载。

在分析方案可行性之前我们现需要了解Windows补丁的结构，才能确定是否可以通过分析补丁定位漏洞并自动修复怎样利用补丁。
下图是从微软security官网上爬去下来的数据绘制成的图片，纵坐标时不同版本windows在各年度发布的补丁数量，此图最初时是为了证明这项工作的意义。

![bug-years](/img/in-post/post-patch-repair/bug-years.png)

## Windows补丁类型

现在微软Windows补丁包中的更新文件大致包含了两类。一类叫做GDR(普通分发版本)，一类叫做QFE(快速修补工程更新)。其中，GDR文件经过了大量严格的测试，稳定性很高。而对QFE所做的测试相对则要相对少一些，所以稳定性亦要低一些。微软的补丁包也按此分为两类: 一类就是安全修补程序，这类补丁包中同时包含了GDR和QFE版本的更新文件，也就是两个副本。微软的很多关键性安全补丁就属于此类。还有一类叫做修复程序，仅包含了QFE版的更新文件。常见的就是一些需要正版验证的补丁。

　　那么为什么安全修补程序要包含两种版本的文件呢?如果你要在系统中安装修复程序，也就是说要安装QFE更新文件。然而当前系统中需要被替换的文件为GDR版，而且版本号要比补丁包中的QFE文件版本号高，那么就不能用补丁包中的QFE文件来替换，而需要用与当前GDR文件版本相同的QFE文件来修补。那么到哪里取得这个文件呢?其实这个QFE文件在你以前安装GDR版更新文件(就是当前系统中使用的文件)时就已经被同时复制到了你的硬盘中。这就是安全修补程序需要同时包含GDR和QFE更新文件，且两类文件版本号都相同的原因。

## 静态分析

补丁的扩展名通常.msu，其本质上就是一个压缩包，压缩包里面包含了四个元素，可以用Wusa解析该文件：

- Windows Update metadata, which describes the update package
- One or more .CAB files that store the update data for the package
- An .XML file which describes the contents of the MSU file
- A properties file that is read by Wusa.exe

![package](/img/in-post/post-patch-repair/update-metadata.png)

我对这四类文件进行了检查分析：

properties.txt主要是被用来判断是否适用于当前系统版本，同时对系统进行自检，看对应问题点是否已经处理。

.xml文件中的action定义了该msu指定系统对update package文件的操作，location定义了补丁安装文件“x86.cab”在msu释放后所在的位置和名称。

![package](/img/in-post/post-patch-repair/wusa.png)

官方说WSUSSCAN.cab是微软用来同步更新文件的东西，那么具体保存的是什么呢，我对三个补丁中解压出的wsussan.cab文件使用二进制文件比较工具进行比对，只有package.xml文件中的package-id,creationdate,updateid等信息发生了改变，可以判断其中保存的微型更新数据库应该是固定不变的，只保存了04-07\10年的更新。