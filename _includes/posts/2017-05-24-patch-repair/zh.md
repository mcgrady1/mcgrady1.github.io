> 2015年时和一个老师讨论过一个题目，大意是使用新版本系统的补丁自动化修复老本系统的bug，这个大背景是老的系统由于官方原因不再提供维护和补丁，但是其和新版本的内核有很多的共用。下面的资料就是在做可行性分析的时候整理的，同时还有部分实验结果。
>
> 以下只用作自我反思，请勿转载。

下图是从微软security官网上爬去下来的数据绘制成的图片，纵坐标时不同版本windows在各年度发布的补丁数量，此图最初时是为了证明这项工作的意义，很有讽刺意味的一件事。

![bug-years](https://github.com/mcgrady1/mcgrady1.github.io/tree/master/img/in-post/post-patch-repair/bug-years.png)