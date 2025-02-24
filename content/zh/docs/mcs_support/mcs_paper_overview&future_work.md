---
title: "MCS综述及rel4的MCS不足"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 对于MCS的研究综述

对于最新的非server（对于server的研究应该已经销声匿迹了吧？可以去研究Sporadic server先）的MCS的相关研究，需要追溯到2007年Vestal [1]的论文。在该论文中提出了多关键系统的概念。虽然这篇论文当中分析的只是单核情况下，固定优先级（fixed priority，FP）的情况。但是却启发了后续的诸多分析（后续诸多声称自己是MCS的模型，大多都是基于此）。

简单来说一下在这种情况下的MCS的模型，对于一个系统中，包括若干个Component，这些Component每个都有一组task set，MCS的其中一个目标就是多个Component之间的运行互不干扰。每个task τi，在不考虑多个关键等级（criticality level）的情况下，应当具有以下几个要素

	Ti : 最小到达间隔周期 （可以认为高于这个到达频率，那么任务将无法处理）

	Di ：Deadline （对于这个要素，在周期性任务中，显然Ti和Di应当相同，但是在零星任务中，则并不总是这样）

	Ci ：computation time，运行时间（应当是说在这个最小到达间隔周期之内的运行时间，但是你非要把这个Ci理解为WCET，也应该问题不大）

	Li ：criticality level 关键性水平。

这些task的期望是不要错过ddl，也就是Di

按照vestal的观点，WCET在不同的关键性水平上并不一样，所以为了分析这些情况下的临界性问题，提出的模型把task的要素从（Ti，Di，Ci，Li）转变为（vector（Ti），Di，vector（Ci），Li），也就是在不同的临界水平，一个task具有不同的最小到达间隔周期和允许他运行的时间长度。

显然在更高的关键等级上，Ci会更多，也就是高关键性任务给他更多的运行时间。带来的结果是，可能有更小的到达时间间隔。

对于这个模型的扩展，以下几种：

1、双临界模型，也就是，只考虑HI和LO两种临界级别的情况。这是最经典的情况，在此基础上衍生出非常多的相关论文，这在后面会分别进行介绍。但是简单来说，上述的task的要素模型变为，（Ti，Di，Ci（HI），Ci（LO），Li）（注：Ti不用HI和LO）。

在此基础上，一个常见的系统模型是这样的，一个系统会以多种临界模式进行运行（在这里就是两种临界模式），一开始按照低临界模型（以及设置的对应的task的Ci（LO）进行安排和调度），在某种情况下，低临界模型下某个任务的临界性不满足（也就是某个高临界任务错过了ddl了），那么就转换到高临界模型（在这种情况下，给高临界任务更多的预算）。至于转不转回低临界模式，有的转回，有的不转。

2、多临界模型[3][4][5]，可以允许若干个临界级别的存在（一种常用的设置是5级[2]，因为在航空界通常使用5级，用于描述不同任务的）。

3.pWCET[6]：多临界模型的Ci其实也是离散的，而这里改成概率分布的（或者起码是大量离散值），当然，更为经典的描述还是说，在置信区间为某个值（比如99%置信区间）的时候，WCET不超过某个时间长度。在此基础上做出相应的分析[7][8]

除此之外，在Vestal的2007年的这篇论文中，Audsley[10]的方法被认为是最优的，RM（rate monotonic）和deadline monotonic（应该就是指EDF），都不是最优的。	

Vestal的后续论文[9]，对上一篇论文进行了模型的概括，以及包括了一个重要的结论，EDF does not dominate FP when criticality levels are introduced,并且一些可行的系统不能被EDF所实现。

2010年，Dorin[11]证明了上述Vestal的框架，比如关于Audsley的方法最优的情况。（应该是说固定优先级的情况下）。

在此基础上，2011年[12]这篇论文基本统一了单核固定优先级下的分析框架。

同样的，对于EDF的调度（基本上是动态优先级的情况）。最早还是Vestal[9]在2008的这篇论文。
除此之外，一个值得一读的框架是EDF-VD方法[13][14]（EDF with Virtual deadlines），有理由相信，这里的虚拟截止时间和EEVDF里面对于虚拟截止时间的探讨是有一定关系的。并且该框架在之后的多个地方，都是当做一个baseline来看待的。

当然，还有一些更扩展的讨论，比如说，处理器的频率是不固定的，还需要考虑功率等等问题。

另一个讨论是，一个任务的WCET通常难以确定，但是semi-clairvoyance的方法[15]认为，可以在一个任务到达的时候告诉系统在不同临界水准的一个WCET，具体到这篇论文，他设置的也是HI和LO两级调度模型，然后把在系统具有在作业到达时确定是否超过LO的运行时间的系统称为semi-clairvoyance的系统。

另外，需要考虑到多级调度的情况，[17]指出多级调度会导致性能下降的问题。而更重要的是，他们提出了scheduling context的概念并应用于MCS[18]（之所以提到这两个，是因为他们也许跟sel4的schedule context的概念相关）

对于更新的论文，在上述大佬们的基础上，更多的是考虑多核的情况。

对于多核情况下的工作。最早是来自于[16]，考虑到这是一个2010年的工作，他使用5级的调度模型，是可以理解的。在最高关键级别的A level，用percpu的cyclic executive。B level同样是percpu的但是基于P-EDF，而C 和D level则是Global的EDF，最后E则是尽最大可能交付的。

而更细化来说，对于多核情况下，主要是任务的分配，可以分为以下几类：

1、global分配

2、partitioned 分配

3、semi-partition 分配

4、其他

对于上述的不同分配方案及其分析不做更多了解，因为很多都基于前面提到的EDF-VD 以及AMC等分析框架。事实上，在近十年的工作中，多核部分会更多一些。
值得一提的是第四项的其他一列，可以看到fluid task model的引入[19]，该模型的设计是以与其利用率成正比的速度执行每项任务。并且被认为如果忽略一个切片统计的任务的运行的话，应当算作这种情况下的最优的多核调度算法了。
除此之外，[20]也应该算是值得阅读的论文，大概来说是可以基于静态表的方法去实现，当然还有其后续，包括tree，DAG（有向无环图）等方法。
	

# rel4的MCS的不足

在微内核下做MCS调度，[21] 这篇论文（注：就是Eurosys18‘，rel4的MCS的提出的那篇论文）给我们在微内核下实现MCS提供了思路和借鉴，但是，现有的实现仍然有一些不足。

具体来说：

1/把Sporadic server的策略放到了内核中，而这和微内核所期望的策略与机制分离本身是相违背的。
尽管按照这篇论文的说法，基于一个SS的静态优先级调度算法，可以在用户态实现一些P-EDF之类的动态优先级的调度算法（反之则不行），但这么做是一个两级的调度策略，按照上面说的论文[17]，这可能并不一定是最优的。与之相对应的，在宏内核的最新论文中，比如ghost等用户态的调度程序[23]，也试图完成类似于微内核的用户态实现用户态调度的策略。

2/sel4实现的MCS调度，所基于Sporadic server的算法，来自于2010年的[22]这一篇论文。而近年来对于MCS的研究，正如前面所说，大部分已经不再关心基于server的调度算法。现有的sel4的MCS的代码，同样难以满足一些更新的问题。比如Vestal提出的多临界性的需求，处理器速度不恒定的问题，还有多处理机情况下的调度——这也是为什么在[21]那篇论文中，使用P-EDF试图说明他的框架可以用于不同的策略中，但是考虑多核之后，现有sel4的核心之间的MCS框架是难以完全适应的。

3/现有的MCS框架，还有一个缺点就是，对于代码的巨大的侵入式修改。这显然不够简单且直观。当我们试图实现一个组件化的操作系统内核，我们希望可以把调度策略单独拿出来，无论这个策略运行在内核态或者用户态。只有这样，我们才可以用不同的调度组件来相互替换。

那么，为什么不直接使用ghost等用户态的调度方案？因为微内核有自己的特点（甚至ghOSt的标题都指出自己是Linux的调度）
首先，比如ghost，他获取调度策略依赖的信息，需要通过内核和用户态调度方案的通信，比如消息队列。但是微内核下，他天然具有更快的通信方式，几乎所有的syscall接口都是内核与用户态、用户进程与用户进程之间的通信表达，这使得两者有形式上的巨大差异。
其次，在我们的调度策略中需要考虑MCS各种实时性的因素，但是其他的用户态调度策略不太考虑这些策略。事实上，用户态的调度进程的切换等时延，也同样需要考虑在内。


# reference

（其实需要看的论文实在是挺多的，这里列举的是最必要的那些了）

1.preemptive scheduling of multi-criticality systems with Varying Degrees of Execution Time Assurance, Vestal, 2007

2.Extending mixed criticality scheduling, 2014（没啥需求的话建议不看）

3.State-based mode switching with applications to mixed criticalit systems（没啥需求的话建议不看）

4.Bounding and shaping the demand of generalized mixed-criticality sporadic task systems（没啥需求的话建议不看）

5.Shedulability analysis of a graph-based task model for mixed-criticality systems（没啥需求的话建议不看）

6.Wcet analysis of probabilistic hard real-time system, 2002

7.PROARTIS: probabilistically analyzable real-time systems（没啥需求的话建议不看）

8.Static probabilistic timing analysis for real-time systems using random replacement caches, 2015

9.schedulability analysis of sporadic tasks with multiple criticality specifications 2008

10.On priority assignment in fixed priority scheduling, 2001

11.Schedulability and sensitivity analysis of multiple criticality tasks with fixed-priorities, 2010

12.Response-time analysis for mixed criticality systems

13.Preemptive uniprocessor scheduling of mixed-criticality sporadic task systems,2015

14.Mixed-criticality scheduling of sporadic task systems

15.Semi-Clairvoyance in Mixed-Criticality Scheduling

16.Mixed-Criticality Real-Time Scheduling for Multicore Systems， 2010

17.flattening hierarchical scheduling，2012

18.On the expressiveness of fixed priority scheduling contexts for mixed criticality scheduling, 2013

19.MC-fluid: Multi-core fluid-based mixed-criticality scheduling， 2014

20.certification-cognizan time-trigged scheduling of mixed-criticality systems

21.Scheduling-context capabilities: a principled, light-weight operating-system mechanism for managing time

22.Mixed-Criticality Real-Time Scheduling for Multicore Systems

23.ghOSt Fast & Flexible User-Space Delegation of Linux Scheduling
