> 2017年初就开始有想法想要把Deep Learning(主要是RNN)和Fuzzing相结合，因为文件格式中的字节序列有一定的时序关联性，和RNN的模型很吻合，训练模型使之能够自动生成有效的种子文件本就是一件很有趣的事情。但因为种种原因，一年多过去了，看着自己的想法，和别人讨论过的思路，或者自己正在做的工作，一个个被别人先实现并发表论文，心里多少有些不好受，但终究是要向前看的，所以今天就把目前已有的论文全部进行整理总结，也希望可以给自己新的启发。
>
> 本文仅供自我反思，请勿转载。

Deep Learning因为近些年硬件计算能力的大幅提升，势头大热，researcher们也拼了命的把自己的研究方向往上面靠，试图分一杯羹。2017年初开始，Fuzzing结合Deep Learning的文章就如雨后春笋，先后在ASE, FSE, ICSE, ISSTA上面发表，虽然我不是做软工的，但也会关注一下这些会议，毕竟安全顶会上的落选文章通常会在这些地方出现。

说实话，虽然我自己也在做这方面，但我一直都不确定Deep Learning是否能真的帮助到Fuzzing，理想很美好，但有很多点在我看来，即使他们的结果还不错，也无法是我信服， 一是他们都没有尝试解释模型学习到的哪些内容在知道Fuzzing发挥效果，二是我有足够的理由相信这些结果可能根本就不是由于他们的本意造成的，可能只是不相关的其他方面的影响。下面我们就先来解析一下这些文章.

## Learn&Fuzz

这篇文章发表在[ASE](https://arxiv.org/pdf/1701.07232.pdf)，其中一个作者是Patrice Godefroid,做为智能模糊测试的开山鼻祖，相信很多人跟我一样，是从他的DART，SAGE，Micro Execution一路读过来的，他的论文影响了我对与Smart Fuzzing最初的认知。另一个作者Rishabh Singh，我看了他最近发表的论文，好像集中在代码自动生成，自我感觉是挖了一堆坑等着年轻人们跟着跳。

最早在16年底我就和一个同学讨论过用RNN自动学习文件格式并生成test case的可能性，当时举得例子恰巧就是这篇文章中用到的pdf格式。我们当时讨论的结果是可能可行，但是一个关键问题没有办法解决，如何让model建立数据对象和索引表之间的关系？数据格式类似一个小型的文件系统，索引指针通常用来帮助对象定位加载地址的关系，不同于图像处理中的时间和空间关系，这种隐性的控制流关系，就我当时的认知，没有模型能够学习到这一层关联。当然还有其他问题，比如sanity，length字段，当然这些都属于控制字段。也正是因为这些问题，我们没有继续往下做。

所以当1月份看到这篇文章的时候我很兴奋也很期待，主要想看大佬们如何解决我们认为的困难题，但读过之后很失望，我们想看的问题一个也没有解决，这篇文章真正学习的实际上是对象内部的数据关联，更直接点说，就是数据段中字符之间的关系，和普通的文本自动生成实际上没有什么太大的区别，建立索引表还是需要先验知识，然后自己写脚本自动生成。实验结果也验证了我们的看法，效果并不好，只变换数据段很触发新的路径和有用的bug，特别是pdf这种展示性的文件格式，只要数据控制在可解析的范围内都能够正常输出。可能真正增加覆盖的也就是不同对象的相互组合，但这也算不上learn&fuzzing，用*Peach*一样可以做到。

## (RNN|DNN|CNN)&Fuzzing

这篇[文章](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/11/neural-fuzzing-mcr.pdf)也是Patrice Godefroid的工作，不得不佩服其占坑的能力。该文章的想法是使用RNN训练模型使之预测数据序列的heatmap，在我看来，所谓heatmap就是一串bytes序列对应的权重序列。举个例子，我们选择文件的前48个bytes被划分为一个序列，其中序列的前四个字节是文件的magic bytes，这样其对应的heatmap就应该是[1,1,1,1,0,...,0]，因为我们修改magic bytes的时候一定会影响程序的执行路径和覆盖率。

在构造training set的时候他们如何界定一个byte或者bit的heatmap为1或者0呢，当对一个文件做bitflip时，如果bit位翻转改变了bitmap，就认为这个bit的heatmap是1。虽然文中用了很多看似复杂的公式，其实意思就是这么简单，x是seed，y是heatmap，y只包含那些可以产生新覆盖的mutation。

在处理数据时候，作者将文件进行了等长切分，比如以64或者128各bytes为单位切分training set中的所有文件，按照顺序依次送入model进行学习，当然我们在实际重现的时候发现这种方法的问题是前期获取training set的时间需要很久，因为一个seed所能提供的正向激励部分很少，整个heatmap是一个稀疏矩阵，大部分的序列都是全零。预测的时候采用同样的方法，对输入文件进行切分，然后根据heatmap对每一部分进行预测，判断需要mutation的偏移。

个人感觉这种方法其实比前一篇文章靠谱很多，但一直没有发出来。

另一篇[博客](https://www.microsoft.com/en-us/research/blog/neural-fuzzing/)提到了微软的DNN方法，但只限于誓言结果，但并没有提到技术细节。

还有一篇七月份最新出的[文章](https://arxiv.org/pdf/1807.05620)，这篇文章和我上个月的时最接近的，或者说是完全一致，本意就是用seed来预测bitmap，从而对种子文件进行筛选。这篇文章我再仔细阅读后我会更新理解，毕竟对于竞争对手不能太大意。

## GANs&Learning

[Faster Fuzzing: Reinitialization with Deep Neural Models](https://arxiv.org/abs/1711.02807), 这篇文章应该算比较早使用GANs model来生成test cases的方法，我记得它最开始的名字不是这个。很巧合的是我们从17年三月份左右就开始讨论用GANs来做点事情，三个人坐那侃了一下午，也没讨论出一个具体的方法。GANs的本意是根据已有的cases生成尽可能有差异的新cases，但和图片应用场景不同的是，图片可以只用像素做训练集，文件则不能只用数据段，我们也不能把数据段单独拿出来训练，就如前面所说的，如果忽略格式的其他控制端只训练数据段，很难触发数据处理以外的分支，而那些分支通常才是crash出现的高发地。但可能优势因为把问题看得太复杂，耽误了太多的时间。

不久前又出现了一篇类似的文章SmartSeed，一堆数据抛进去，挑一个收敛速度不错的mGANs model，找到一些crash，这篇文章这几句话差不多就可以概述了。就通过文中的描述，我可以断定着又是一片文本自动生成的工作，这个自动生成论文没什么区别。

就像一个同学说的，对于一个fuzzing相关的工作，reviewers根本就不看你整出什么有理论依据的方法，aflfast中理论忽悠极强的除外，找到crash就收。看着github上面从未分析的数百个crash，我愧对你们了。

## Reforcment Learning&Fuzzing

issta 2017上的一篇文章，大家可以自行搜索，其主要讲的是使用强化学习处理持续更新环境下的测试问题，这篇文章算是这里面我看的最认真的，因为强化学习也是我之前想要结合的一个技术，但一直苦于没有具体的问题，这篇文章解决问题的出发点非常好。

持续更新(CI)是工业生产中常见环境，其是为了保障软件的功能完善和安全性而存在的，持续更新环境包括了版本控制，软件配置管理，自动编译，软件新释放版本的自动化测试。这里主要瞄准的就是对与软件新释放版本的自动化测试优化。在该问题中，如何自学习选择并运行合适且与程序高度相关的种子。

准确的说，对于CI而言，测试人员希望工具不要一直集中在浅层逻辑，而是尽可能的接近新添加的代码功能，但传统方法的测试用例通常是根据软件的全局功能测试的，主要解决全局覆盖率的问题，而不是倾向于单元测试用例。所以对于CI而言，主要面对的问题就是种子选择：

- 优先选择，历史方法是基于历史的优先原则，历史上使系统失效的测试用例之后更有可能使之出现问题。
- 种子选择，CI系统也需要面对一个问题，因为面对着上线压力，所以必须在有限时间内选择一定数量的测试用例，而不是全部执行。

以上两个问题使用固定策略的测试系统都不能很好的解决，因为无法结合新系统的功能特性。本文设计了一个强化学习网络去学习种子选择和排序策略，基于其之前的运行结果，该模型趋向于之前成功检测到异常的测试代码段。

#### 解决方案模型 

本文选择model-free和online的模型，model-free意味着开始的时候并不知道环境是怎样的，也不知道你的操作会怎样影响环境，这很方法适合种子选择和排序策略的问题。online意味着持续学习，因为软件的修改和相关环境的变化都会影响我们的模型，所以选择在线模型。在RL中，agent接收环境的状态state，并选择合适的操作action，这个action可以通过学到的策略，或者是随机策略，agent收到反馈对其action进行评估。

#### Reward函数设计

三个方法计算reward

- 失败数量，测试一个种子时运行出现错误的用例数量。RL agent的目标就是失败用例最大化。
- 失败用例，每个case 的verdict值作为其自己的reward，如果一个测试用例顺利的执行，则其不会对agent产生reward。每个测试用例包含一个决策值verdict和一个持续时间duration，只有在执行完这个集合后才能得到这两个值。如何测试用例顺利结束，则verdict等于1，出现异常或没有执行为0。

以上两个reward隐性的引导agent优先选择导致失败的测试用例。这样做会有问题，可能使得agent挑选的用例都测试趋向同一个函数。

- 时间排序，对每种排序所用的时间进行reward，如果导致失败的测试用例运行的数量多则分数高。

总的来说，以上的reward策略设计的并不适用于所有场景？使用失败的测试用例来排序很容易让测试代码陷入对某个程序块的测试，但是对于这种基于补丁或者持续系统而言，这个方案是可行的。

#### Action Selection

取决于agent使用的策略，这里使用的方法近似于优化问题。开始policy是空白的，之后趋向于选择之前执行的用例中reward value较高的用例。

总的来说 agent的state是测试用例集, action是各个种子的优先级，这可以使用多个函数进行近似，这里作者主要使用了两种函数近似方法。第一个是tableau， 它包含两个表来追踪state和需要选择的action。一个表记录了每种state下不同action被选择的频率。另一个表记录了每种action平均收到的奖励。这样通过这两个表就可以选择期望值最高的action作为当前的行为。第二个就是ANN，其实就是神经网络。网络的输入是种子文件，输出被解析为优先级。 

PG的那一个强化学习的文章就不评论了，实验数据都没有，方法也很老套，着实有点糊弄人了。


## Direction of Deep Learning&Fuzzing？

- 我们使用有限的计算资源和时间想得到最好的fuzzing效果，这就需要好的决策帮我们进行种子的选择，等价于决策问题？

- 学习软件的组件行为：理解应用软件对于不同的输入所做出的输出的差异。

- 单纯的暴力测试，或者是简单的概率选择+暴力测试在实际应用中效果就不错，但不管怎样这种方法还是相对原始。暴力测试的可以用actor-critic算法进行替换，或者说是强化学习。 

  “actor”是fuzzer, 其尝试生成应用程序的模糊输入。“critic”是分析器，它负责检查应用程序的状态 - 有多少次崩溃，有多少次被覆盖，有多少内存被使用等等状态信息，我们先尝试只用bitmap来表示状态。

-  数据语料库-这种方法对于暴力测试会有效果改善，但对于主动性的测试不一定好用，因此，在fuzzing中，语料库并不会太有用，语料库越大说明你生成的样本和实际得到的样本越多，这都是需要消耗更多的时间。也就说model已有的知识实际上都已经被测试过，再拿来能给我们带来什么优化呢。

-  pattern提取，对于造成覆盖率增长，程序崩溃，内存占用极大的样本进行patter提取，并以此来做patter-feedback的fuzzing测试。特别需要注意的是，这些pattern可能和不同的api有直接的对应关系。

   ​

   ​

